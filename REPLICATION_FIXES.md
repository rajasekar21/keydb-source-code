# KeyDB Replication Risk Fixes Release Notes

**Date:** 2026-05-05
**Branch:** main
**Scope:** Replication subsystem risks identified by production-grade deep analysis

---

## Overview

This release addresses three bugs in KeyDB's replication subsystem:

- **REPL-01**: Inconsistent memory-ordering on `repl_lowest_off` stores/loads — direct
  assignment operator used in `feedReplicationBacklog` where `.store(release)` is required
  for consistency with the established acquire/release pairing elsewhere.
- **REPL-02**: Data race on `repl_curr_off` between the REPLCONF ACK handler and the
  concurrent threadsafe `writeToClient` path — missing `repl_backlog_lock` acquisition.
- **REPL-03**: Integer overflow in the PSYNC partial-resync boundary check —
  `repl_backlog_off + repl_backlog_histlen` overflows when the master has been running
  long enough that the offset approaches `LLONG_MAX`.

---

## REPL-01 — `repl_lowest_off`: Inconsistent Atomic Memory Ordering

**File:** `src/replication.cpp` — `feedReplicationBacklog()` (lines 405, 407, 421) and
`trimReplicationBacklog()` (line 5846)

### Problem

`g_pserver->repl_lowest_off` is declared as `std::atomic<long long>`. Every write in
the REPLCONF ACK handler (our PERF-04 fix) and in `replicationSetupSlaveForFullResync`
uses explicit `.store(value, std::memory_order_release)`, and every read in
`feedReplicationBacklog` uses `.load(std::memory_order_seq_cst)` — establishing a clear
acquire/release pairing.

However, three stores inside `feedReplicationBacklog` and one load in
`trimReplicationBacklog` used the implicit `operator=` / implicit-conversion forms:

```cpp
// feedReplicationBacklog — implicit operator=, uses seq_cst instead of release:
g_pserver->repl_lowest_off = -1;        // line 405
g_pserver->repl_lowest_off = min_offset; // line 407
g_pserver->repl_lowest_off = -1;        // line 421

// trimReplicationBacklog — implicit conversion, uses seq_cst instead of acquire:
if (g_pserver->repl_lowest_off > 0 && ...)  // line 5846
```

`operator=` on `std::atomic<T>` maps to `store(seq_cst)`, and the implicit conversion
maps to `load(seq_cst)`. These are *stronger* than the release/acquire used elsewhere —
they are not incorrect on their own — but they break the consistent memory-ordering
contract that the rest of the codebase maintains. An `seq_cst` store on one thread pairs
with an `seq_cst` load on another, not with a `memory_order_acquire` load, which can
cause the compiler or CPU to reorder surrounding non-atomic accesses in ways that were
not intended.

### Fix

Replace all implicit atomic operations with explicit ones matching the established
`memory_order_release` / `memory_order_acquire` pattern:

```cpp
// feedReplicationBacklog — stores:
g_pserver->repl_lowest_off.store(-1, std::memory_order_release);
g_pserver->repl_lowest_off.store(min_offset, std::memory_order_release);

// trimReplicationBacklog — load:
if (g_pserver->repl_lowest_off.load(std::memory_order_acquire) > 0 && ...)
```

### Impact

| Scenario | Before | After |
|---|---|---|
| Optimizer/CPU reordering around repl_lowest_off stores | Possible under acquire/release model | Eliminated — consistent release stores |
| Code review clarity | Implicit ops look like plain assignments | Intent is explicit |

---

## REPL-02 — REPLCONF ACK Handler: Data Race on `repl_curr_off`

**File:** `src/replication.cpp` — `replconfCommand()` (~line 1717)

### Problem

`repl_curr_off` is the write-progress cursor for each replica client. It is updated in
the replica-specific `writeToClient` path under `repl_backlog_lock`:

```cpp
// networking.cpp ~line 1842-1871 (writeToClient for replicas):
std::unique_lock<fastlock> repl_backlog_lock(g_pserver->repl_backlog_lock);
// ...
c->repl_curr_off += nwritten;    // written under repl_backlog_lock
```

The threadsafe write path can invoke `writeToClient` from a background I/O thread
**without holding `g_lock`**. Meanwhile, the REPLCONF ACK handler runs under `g_lock`
and scans all replicas to compute `repl_lowest_off`, reading each replica's
`repl_curr_off` **without holding `repl_backlog_lock`**:

```cpp
// replication.cpp ~line 1713-1718 — NO repl_backlog_lock held:
while ((ack_ln = listNext(&ack_li))) {
    client *replica = (client*)listNodeValue(ack_ln);
    // ...
    new_min = std::min(new_min, replica->repl_curr_off);  // DATA RACE
}
```

Because the writer holds `repl_backlog_lock` (not `g_lock`) and the reader holds
`g_lock` (not `repl_backlog_lock`), both can execute concurrently. A torn 64-bit read
of `repl_curr_off` causes `repl_lowest_off` to be computed from a partially-updated
value, which can make the backlog appear larger or smaller than it is — potentially
causing premature replica disconnection or incorrect backlog trimming.

### Fix

Acquire `repl_backlog_lock` for the duration of the scan in the ACK handler. This lock
is already held by the writer path that updates `repl_curr_off`, and the ordering
`g_lock → repl_backlog_lock` is already established throughout the codebase (e.g.,
`feedReplicationBacklog` acquires both in this order). No new lock dependencies are
introduced:

```cpp
{
    long long new_min = LLONG_MAX;
    listIter ack_li;
    listNode *ack_ln;
    std::lock_guard<fastlock> backlog_guard(g_pserver->repl_backlog_lock);
    listRewind(g_pserver->slaves, &ack_li);
    while ((ack_ln = listNext(&ack_li))) {
        client *replica = (client*)listNodeValue(ack_ln);
        if (!canFeedReplicaReplBuffer(replica)) continue;
        if (replica->flags & CLIENT_CLOSE_ASAP) continue;
        new_min = std::min(new_min, replica->repl_curr_off);  // now race-free
    }
    g_pserver->repl_lowest_off.store(
        (new_min == LLONG_MAX) ? -1 : new_min,
        std::memory_order_release);
}
```

### Impact

| Scenario | Before | After |
|---|---|---|
| `repl_curr_off` torn read during concurrent I/O | Possible on any SMP system | Eliminated |
| False premature replica disconnect | Possible (overstated backlog need) | Eliminated |
| Incorrect backlog trimming | Possible (understated backlog need) | Eliminated |
| Lock contention | — | Minimal: backlog lock held ~microseconds per ACK |

---

## REPL-03 — PSYNC Boundary Check: Integer Overflow in Backlog Range Validation

**File:** `src/replication.cpp` — `masterTryPartialResynchronization()` (~line 941)

### Problem

When a replica reconnects and sends `PSYNC replid offset`, the master validates that the
requested offset falls within the retained backlog window:

```cpp
if (!g_pserver->repl_backlog ||
    psync_offset < g_pserver->repl_backlog_off ||
    psync_offset > (g_pserver->repl_backlog_off + g_pserver->repl_backlog_histlen))
```

`repl_backlog_off` is approximately `master_repl_offset - repl_backlog_histlen`.
`repl_backlog_histlen` can be up to `repl_backlog_size` (configurable up to gigabytes
and in practice several hundred megabytes on busy masters). After extended uptime —
roughly when `master_repl_offset` approaches `LLONG_MAX / 2` (about 4.6 × 10¹⁸ bytes,
equivalent to ~150 years at 1 GB/s) — the addition wraps around:

```
repl_backlog_off + repl_backlog_histlen → wraps to negative or very small positive
```

This makes the upper-bound check `psync_offset > (wrapped_negative)` always false,
allowing **any** `psync_offset` to pass the PSYNC validation. A replica with a stale
offset that should have triggered a full resync instead receives a slice of the circular
backlog starting at an incorrect position, causing **silent data corruption** on the
replica.

### Fix

Replace the addition-based upper bound with `master_repl_offset`, which is semantically
identical (the backlog covers exactly `[repl_backlog_off, master_repl_offset]`) and
never overflows because `psync_offset` and `master_repl_offset` are both bounded by the
same 64-bit range:

```cpp
/* Use master_repl_offset as the upper bound instead of
 * repl_backlog_off + repl_backlog_histlen: the two are semantically
 * equivalent (the backlog covers exactly [repl_backlog_off,
 * master_repl_offset]), but the addition can overflow when both
 * operands are near LLONG_MAX on a long-running master. */
if (!g_pserver->repl_backlog ||
    psync_offset < g_pserver->repl_backlog_off ||
    psync_offset > g_pserver->master_repl_offset)
```

### Impact

| Scenario | Before | After |
|---|---|---|
| Master uptime > ~150 years at 1 GB/s repl throughput | Overflow → silent data corruption | Correct full-resync forced |
| Master offset near LLONG_MAX (e.g., benchmark or large datasets) | PSYNC accepts invalid offsets | Rejected, full resync triggered |
| Normal PSYNC with valid offset | Unaffected | Unaffected |

---

## Files Changed

| File | Change |
|---|---|
| `src/replication.cpp` | REPL-01: Replace implicit atomic ops with explicit `.store(release)` / `.load(acquire)` on `repl_lowest_off` |
| `src/replication.cpp` | REPL-02: Acquire `repl_backlog_lock` in REPLCONF ACK scan of `repl_curr_off` |
| `src/replication.cpp` | REPL-03: Replace `repl_backlog_off + repl_backlog_histlen` with `master_repl_offset` in PSYNC boundary check |

---

## Validation

```bash
# REPL-01: Verify consistent atomic ordering — grep confirms all stores use .store():
grep -n "repl_lowest_off" src/replication.cpp
# Expected: all writes use .store(, std::memory_order_release)
#           all reads  use .load(std::memory_order_acquire) or .load(std::memory_order_seq_cst)

# REPL-02: Confirm lock is held during repl_curr_off scan
# Start a 2-node setup (master + replica), run a write flood, check for replica disconnects:
redis-benchmark -p 6379 -t set -n 5000000 -d 512 -q &
redis-cli -p 6380 info replication | grep -E "master_link_status|lag"
# Expected: master_link_status:up throughout; no spurious disconnects in server log

# REPL-03: Verify PSYNC rejects offset > master_repl_offset
# Simulate stale PSYNC by sending a manually crafted offset above the master's:
redis-cli -p 6379 DEBUG SET-ACTIVE-EXPIRE 0
current_offset=$(redis-cli -p 6379 info replication | grep master_repl_offset | cut -d: -f2 | tr -d '\r')
# Send PSYNC with offset 1 above current (should full-resync):
redis-cli -p 6379 --no-auth-warning -3 PSYNC "$(redis-cli -p 6379 info replication | grep master_replid | head -1 | cut -d: -f2 | tr -d ' \r')" $((current_offset + 1))
# Expected: +FULLRESYNC response
```
