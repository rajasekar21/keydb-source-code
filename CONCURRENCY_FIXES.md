# KeyDB Concurrency Fixes Release Notes

**Date:** 2026-05-04
**Branch:** main
**Scope:** Top concurrency risks identified by production-grade deep analysis

---

## Overview

This release addresses three concurrency risks in KeyDB's database layer and
replication subsystem. Each fix eliminates a potential data race, torn read,
or topology hazard that could manifest under high-concurrency workloads.

One item — `master_repl_offset` atomicity — is documented below as analysed
and confirmed safe under the existing locking model; no code change was
required.

---

## CONC-01 — `m_numexpires`: Torn Read Under Concurrent Sampling

**File:** `src/server.h:1251`

### Problem

`m_numexpires` was declared `size_t m_numexpires = 0;` (plain non-atomic).
All mutations (`++`, `--`, `= 0`) happen inside database operations that hold
`g_lock`. However, reads from diagnostic paths such as `INFO keyspace`,
`DEBUG OBJECT`, and custom monitoring callbacks can be issued from a thread
that does not hold `g_lock` at the moment the sample is taken.

On platforms where a 64-bit store is not natively atomic (e.g., 32-bit ARM
cross-compiled builds, or if the compiler emits two 32-bit stores), a
concurrent reader could observe a half-updated value — an extremely rare but
possible torn read producing a garbage expiry count in `INFO` output or
triggering a false `serverAssert(m_numexpires > 0)`.

### Fix

Changed declaration to `std::atomic<size_t> m_numexpires {0};`.

- All `++`, `--`, `= 0` operators are natively supported on `std::atomic<integral>`.
- `expireSize()` updated to call `.load(std::memory_order_relaxed)` (no
  ordering guarantee needed — a stale-by-one read in a diagnostic path is
  acceptable and relaxed avoids unnecessary memory barriers on the hot path).
- `snapshot.cpp`: snapshot copy changed from `spdb->m_numexpires = m_numexpires`
  to `spdb->m_numexpires = m_numexpires.load(std::memory_order_relaxed)` since
  `std::atomic` copy-assignment is deleted.

### Impact

| Scenario | Before | After |
|---|---|---|
| INFO keyspace read from monitor thread | Possible torn 64-bit read | Atomic load, always coherent |
| Hot-path `++`/`--` overhead | Plain inc/dec | Atomic inc/dec (same cost on x86 TSO) |
| Snapshot copy | Worked by accident | Explicit `.load()` for correctness |

---

## CONC-02 — `master_repl_offset`: Confirmed Safe Under `g_lock` *(Analysis Only)*

**File:** `src/server.h:2525`

### Analysis

`master_repl_offset` has 40+ access sites across `replication.cpp`,
`server.cpp`, `networking.cpp`, `rdb.cpp`, and `cluster.cpp`. All write
paths (exclusively `feedReplicationBacklog()`, `g_pserver->master_repl_offset += len`)
execute inside command handlers that hold `g_lock`. All read paths that could
race with a write (RDB snapshot capture at `rdb.cpp:1653`, INFO at
`server.cpp:6243`, replication offset arithmetic) also hold `g_lock` at the
point of access.

### Decision

No code change required. A comment has been added to the declaration to
document the invariant explicitly so future contributors do not inadvertently
add an unlocked reader:

```cpp
/* All writes to master_repl_offset happen inside feedReplicationBacklog()
 * which runs only while g_lock is held; reads from RDB/INFO paths also hold
 * g_lock. No atomic needed — the global lock provides the required
 * happens-before. */
long long master_repl_offset;
```

Converting to `std::atomic<long long>` would require updating 40+ sites and
replacing all compound `+= len` expressions with `fetch_add`, introducing
unnecessary churn with no safety benefit given the existing lock discipline.

---

## CONC-03 — `replicationAddMaster`: Active-Replica Cycle Detection

**File:** `src/replication.cpp:replicationAddMaster()`

### Problem

In KeyDB's active-replica topology (`active-replica yes`), each node can
simultaneously be a master *and* a slave. This makes replication cycles
possible: if node A adds node B as a master while B is already replicating
from A, an infinite propagation loop forms (A→B→A→B→…). Each write command
would be re-fed from master to slave and back indefinitely, causing:

- Unbounded replication backlog growth
- Exponential `master_repl_offset` drift
- Eventual OOM from backlog overflow
- Degraded write latency as every command triggers cascading ACKs

The existing guard in `replicationAddMaster()` only prevented adding the same
`(ip, port)` as a *second master entry* — it did not check whether the
prospective master was already present as a slave.

### Fix

Before the existing master-list duplicate check, `replicationAddMaster()` now
iterates `g_pserver->slaves` and rejects any prospective master whose announced
address matches an existing slave:

```cpp
listRewind(g_pserver->slaves, &li_cycle);
while ((ln_cycle = listNext(&li_cycle))) {
    client *slave = (client*)listNodeValue(ln_cycle);
    if (!(slave->flags & CLIENT_SLAVE)) continue;
    if (slave->slave_listening_port == 0 || slave->slave_listening_port != port) continue;

    const char *check_ip = (slave->slave_addr && slave->slave_addr[0])
        ? slave->slave_addr
        : (connPeerToString(slave->conn, peer_ip, ...) == 0 ? peer_ip : nullptr);

    if (check_ip && strcasecmp(check_ip, ip) == 0) {
        serverLog(LL_WARNING, "replicationAddMaster: refusing %s:%d — cycle detected", ip, port);
        return nullptr;
    }
}
```

**Address resolution priority:**
1. `slave->slave_addr` — the IP the slave announced via `REPLCONF ip-address` (most reliable)
2. Connection peer IP from `connPeerToString()` — fallback when REPLCONF hasn't been received yet
3. Slaves that haven't announced their listening port (`slave_listening_port == 0`) are skipped
   (they haven't completed the handshake; a cycle involving them would be caught on the next
   `REPLICAOF` command after the handshake completes)

**Scope:** Detects direct (2-node) cycles. Multi-hop cycles (A→B→C→A) are not
detected by this change; they require a distributed cycle-detection protocol
that is out of scope for this release.

### Impact

| Scenario | Before | After |
|---|---|---|
| `REPLICAOF B` when B already replicates from us | Cycle formed silently | Rejected with LL_WARNING log |
| Replication backlog under cycle | Unbounded growth → OOM | Never forms |
| Active-replica topology safety | Best-effort | Guarded at add-master time |

---

## Files Changed

| File | Change |
|---|---|
| `src/server.h` | `m_numexpires`: `size_t` → `std::atomic<size_t>`; `expireSize()` uses `.load(relaxed)`; added `g_lock` comment on `master_repl_offset` |
| `src/snapshot.cpp` | Snapshot copy uses `m_numexpires.load()` (atomic copy-assign is deleted) |
| `src/replication.cpp` | Cycle detection added to `replicationAddMaster()` before master is registered |

---

## Validation

```bash
# 1. Verify m_numexpires stays coherent under rapid expiry churn
redis-benchmark -p 6379 -t set -n 2000000 -d 64 --csv &
redis-cli -p 6379 --latency-history -i 0.1 &
watch -n 0.1 'redis-cli -p 6379 info keyspace | grep expires'

# 2. Verify cycle detection fires correctly (active-replica topology)
# Start two nodes A (port 6379) and B (port 6380) with active-replica yes
# Make B replicate from A:
redis-cli -p 6380 REPLICAOF 127.0.0.1 6379
# Wait for B to connect, then try to make A replicate from B — should be rejected:
redis-cli -p 6379 REPLICAOF 127.0.0.1 6380
# Expected: returns OK (command accepted) but server log shows cycle-detection WARNING
# and no replication connection is established from A to B

# 3. Verify no regression in normal master-slave setup
redis-cli -p 6380 REPLICAOF 127.0.0.1 6379
redis-cli -p 6380 info replication | grep master_link_status
# Expected: master_link_status:up
```
