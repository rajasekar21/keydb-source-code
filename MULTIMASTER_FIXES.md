# KeyDB Multi-Master Replication Fixes Release Notes

**Date:** 2026-05-05
**Branch:** main
**Scope:** Multi-master replication bottlenecks identified by production-grade deep analysis

---

## Overview

This release addresses two bottlenecks in KeyDB's multi-master replication subsystem
that degrade throughput and correctness specifically when more than one master is
configured:

- **MMR-01**: `fInMasterConnection` single-flag pattern in `replicationCron()` serialises
  master connection initiation to at most one new connection per cron tick, causing O(N)
  reconnection time in an N-master topology.
- **MMR-02**: REPLCONF GETACK handler broadcasts acknowledgements to **all** configured
  masters instead of only the master that issued the GETACK, causing O(N) unnecessary ACK
  writes per request.

---

## MMR-01 — `replicationCron`: Serialised Master Connection Initiation

**File:** `src/replication.cpp` — `replicationCron()` (~line 4853)

### Problem

`replicationCron()` used a single boolean flag to prevent more than one master
connection from being initiated per cron tick:

```cpp
bool fInMasterConnection = false;
while ((lnMaster = listNext(&liMaster)) && !fInMasterConnection) {
    redisMaster *mi = (redisMaster*)listNodeValue(lnMaster);
    if (mi->repl_state != REPL_STATE_NONE && mi->repl_state != REPL_STATE_CONNECTED
        && mi->repl_state != REPL_STATE_CONNECT)
    {
        fInMasterConnection = true;   // stops scan early
    }
}
// ...
if (mi->repl_state == REPL_STATE_CONNECT && !fInMasterConnection && ...) {
    connectWithMaster(mi);
    fInMasterConnection = true;       // blocks subsequent masters
}
```

Two effects:

1. **Early scan termination**: The first loop exits as soon as any master is in a
   mid-handshake state, so masters later in the list are never checked for timeouts
   on that cron tick.
2. **One-per-tick connection**: Even if N masters are all in `REPL_STATE_CONNECT`
   simultaneously (e.g., after a network partition recovers), only one is advanced
   per 100 ms cron tick. Reconnection to N masters takes at least N × 100 ms = up
   to seconds of unnecessary delay.

This design originates from the single-master Redis codebase where at most one master
exists; it is not intentional for multi-master KeyDB topologies.

### Fix

Replace the boolean flag with an integer counter that counts masters currently in a
mid-handshake state, then allow up to `MAX_CONCURRENT_MASTER_HANDSHAKES = 4` new
connections per cron tick:

```cpp
static const int MAX_CONCURRENT_MASTER_HANDSHAKES = 4;
int active_handshakes = 0;
while ((lnMaster = listNext(&liMaster))) {           // full scan — no early exit
    redisMaster *mi = (redisMaster*)listNodeValue(lnMaster);
    if (mi->repl_state != REPL_STATE_NONE && mi->repl_state != REPL_STATE_CONNECTED
        && mi->repl_state != REPL_STATE_CONNECT)
    {
        active_handshakes++;
    }
}
// ...
if (mi->repl_state == REPL_STATE_CONNECT &&
    active_handshakes < MAX_CONCURRENT_MASTER_HANDSHAKES &&
    !g_pserver->loading && !g_pserver->FRdbSaveInProgress())
{
    connectWithMaster(mi);
    active_handshakes++;
}
```

The cap of 4 prevents resource exhaustion when many masters need simultaneous reconnect
(e.g., after a partition) while still allowing N-master topologies to recover in a single
cron tick for small N.

### Impact

| Scenario | Before | After |
|---|---|---|
| N masters all need reconnect simultaneously | N × 100 ms serialised delay | ≤ 1 cron tick for N ≤ 4 |
| Master mid-handshake blocks timeout checks on later masters | Yes — early loop exit | No — full scan every tick |
| Resource exhaustion under mass-reconnect | N/A (serialised) | Capped at 4 simultaneous handshakes |
| Single-master topologies | Unaffected | Unaffected |

---

## MMR-02 — REPLCONF GETACK: Broadcast to All Masters Instead of Requester

**File:** `src/replication.cpp` — `replconfCommand()` (~line 1747)

### Problem

When a master sends `REPLCONF GETACK *` to request an immediate replication offset
acknowledgement, the handler broadcast the ACK to **all** configured masters:

```cpp
} else if (!strcasecmp(..., "getack")) {
    listIter li;
    listNode *ln;
    listRewind(g_pserver->masters, &li);
    while ((ln = listNext(&li)))
        replicationSendAck((redisMaster*)listNodeValue(ln));  // ALL masters
    return;
}
```

In an N-master topology, each GETACK from any single master causes N ACK writes:
- O(N) `write()` syscalls on the replication sockets of unrelated masters
- O(N) wakeups of master I/O threads that did not request an ACK
- Incorrect ACK sequencing: masters receive ACKs they did not request, which can
  cause them to advance their `repl_backlog_off` prematurely and trim backlog
  that other, slower replicas still need

The existing `MasterInfoFromClient(c)` helper at line 5367 maps a `client*` to its
`redisMaster*` and is already used elsewhere in the file.

### Fix

Use `MasterInfoFromClient(c)` to send the ACK only to the master that issued the
GETACK, with a fallback to the original broadcast for the (unexpected) case where the
client cannot be mapped:

```cpp
} else if (!strcasecmp(..., "getack")) {
    redisMaster *requesting_master = MasterInfoFromClient(c);
    if (requesting_master != nullptr) {
        replicationSendAck(requesting_master);
    } else {
        /* Fallback: broadcast to all masters (backward-compatible). */
        listIter li;
        listNode *ln;
        listRewind(g_pserver->masters, &li);
        while ((ln = listNext(&li)))
            replicationSendAck((redisMaster*)listNodeValue(ln));
    }
    return;
}
```

### Impact

| Scenario | Before | After |
|---|---|---|
| GETACK from one master in N-master topology | N ACK writes | 1 ACK write |
| Spurious ACKs advancing unrelated masters' backlog | Possible | Eliminated |
| Single-master topology | Unaffected | Unaffected |
| Client not in masters list (unexpected) | N/A | Falls back to broadcast |

---

## Files Changed

| File | Change |
|---|---|
| `src/replication.cpp` | MMR-01: Replace `fInMasterConnection` flag with `active_handshakes` counter; allow up to 4 concurrent handshakes |
| `src/replication.cpp` | MMR-02: Scope REPLCONF GETACK ACK to requesting master via `MasterInfoFromClient(c)` |

---

## Validation

```bash
# MMR-01: Verify parallel reconnect in N-master topology
# Start 3 KeyDB masters on ports 6381, 6382, 6383 and one replica on 6379
# Configure replica with REPLICAOF to all three masters, then kill all three
# simultaneously and restart them. Measure time to full reconnect:
time_before=$(date +%s%N)
redis-cli -p 6381 DEBUG SLEEP 0 &
redis-cli -p 6382 DEBUG SLEEP 0 &
redis-cli -p 6383 DEBUG SLEEP 0 &
# After all three masters restart:
watch -n 0.1 'redis-cli -p 6379 info replication | grep -c master_link_status:up'
# Expected before fix: reconnect completes in ~300ms (3 × 100ms cron ticks)
# Expected after fix:  reconnect completes in ~100ms (1 cron tick for N=3 ≤ 4)

# MMR-02: Confirm GETACK sends exactly 1 ACK in multi-master mode
# Enable debug logging and observe ACK counts:
redis-cli -p 6379 CONFIG SET loglevel debug
# In server log, count "Sending REPLCONF ACK" lines per GETACK:
# Expected before fix: N lines per GETACK (one per master)
# Expected after fix:  1 line per GETACK (only to requesting master)
```
