# KeyDB Critical Bug Fix Release Notes

**Date:** 2026-05-04
**Branch:** main
**Scope:** Critical safety, correctness, and security fixes identified by deep production-grade analysis

---

## Summary

This release fixes **7 critical bugs** spanning memory safety, concurrency, replication,
and crash-handler correctness. All fixes are targeted code-level changes with no
behavior changes to normal operation.

---

## Critical Bug Fixes

### [BUG-01] `freeClientAsync` Lock-Order Deadlock
**File:** `src/networking.cpp`
**Severity:** Critical — Production deadlock

`freeClientAsync()` previously acquired `c->lock` first, then `g_lockasyncfree`.
`freeClientsInAsyncFreeQueue()` acquires them in the opposite order. This ABBA
lock inversion could cause all server threads to deadlock permanently under
concurrent client close and async queue drain operations.

**Fix:** Reversed lock acquisition order in `freeClientAsync()` to always acquire
`g_lockasyncfree` before `c->lock`, matching the order used in `freeClientsInAsyncFreeQueue()`.

---

### [BUG-02] `bulkreadBuffer` Double-Free on RDB Exception Path
**File:** `src/replication.cpp`
**Severity:** Critical — Heap corruption / potential crash

`readSnapshotBulkPayload()` freed `parseState` on the normal completion path but
left `mi->bulkreadBuffer` non-null. If any exception fired after the completion
block (e.g., during RDB validation), `cancelReplicationHandshake()` would be
called again, attempting a second `sdsfree()` on the already-dangling buffer
pointer — causing heap corruption.

**Fix:** Added `sdsfree(mi->bulkreadBuffer); mi->bulkreadBuffer = nullptr;`
immediately after `delete mi->parseState` in the normal completion path.

---

### [BUG-03] BIO Thread `pthread_cancel` While Holding `bio_mutex`
**File:** `src/bio.cpp`
**Severity:** Critical — Deadlock on crash

BIO threads called `makeThreadKillable()` making them async-cancellable. The crash
handler called `pthread_cancel()` on them. If cancellation arrived while a BIO
thread held `bio_mutex[type]`, the mutex was permanently abandoned. Any subsequent
`bioCreateBackgroundJob()` call would deadlock waiting for that mutex.

**Fix:** Replaced `pthread_cancel`-based shutdown with a cooperative `bio_should_exit`
flag. `bioKillThreads()` now sets the flag, broadcasts all condition variables to
wake sleeping threads, and joins each thread — guaranteeing clean mutex release
before the thread exits.

---

### [BUG-04] `g_fInCrash` Signal-Safety Race
**Files:** `src/debug.cpp`, `src/serverassert.h`, `src/fastlock.cpp`, `src/server.h`
**Severity:** Critical — Signal-handler data race

`g_fInCrash` was a plain `int` written from signal handlers and read from deadlock
detector threads without any synchronization. On relaxed-memory architectures, the
deadlock detector could proceed past the `g_fInCrash` guard while a crash handler
was concurrently writing it, causing undefined behavior in the crash path.

**Fix:** Changed `g_fInCrash` to `std::atomic<int>` with `memory_order_acquire`
loads at all read sites and updated all declarations across `serverassert.h`,
`server.h`, `debug.cpp`, and `fastlock.cpp` to reflect the new type.

---

### [BUG-05] Deadlock Detector `fInDeadlock` Static Variable Race
**File:** `src/fastlock.cpp`
**Severity:** Critical — Crash handler re-entrancy

The `static volatile bool fInDeadlock` inside `DeadlockDetector::registerwait()`
was checked and set without atomic operations. Multiple threads detecting a deadlock
simultaneously could both pass the guard, both enter crash-reporting code, and both
attempt to acquire the internal detector lock — causing another deadlock inside the
deadlock detector itself.

**Fix:** Changed `fInDeadlock` to `std::atomic<bool>` with `memory_order_acquire`
loads and `memory_order_release` stores.

---

### [BUG-06] `pool_free` Calls `sfree()` on Pointer of Unknown Allocator Origin
**File:** `src/storage.cpp`
**Severity:** Critical — Heap corruption

When `pool_free()` failed to locate a pointer in any pool page, it fell through to
`sfree(obj)` which routes through `memkind_free()`. If the pointer was not
`memkind`-allocated (e.g., a stale pointer, a stack address, or a pointer from a
different allocator), this would corrupt the allocator's internal metadata with no
diagnostic output.

**Fix:** Replaced the silent `sfree()` fallback with `serverPanic()`. An unknown
pointer in `pool_free()` always indicates a programming error and must surface
immediately rather than silently corrupting the heap.

---

### [BUG-07] PSYNC `+FULLRESYNC` Offset Not Bounds-Checked — Integer Overflow DoS
**File:** `src/replication.cpp`
**Severity:** Critical — Replication protocol abuse / infinite resync loop

The `+FULLRESYNC <replid> <offset>` reply from the master was parsed with
`strtoll()` and stored directly without validation. A malicious or corrupted master
could send `LLONG_MAX` as the offset. Subsequent `master_repl_offset += len`
arithmetic would overflow from positive to negative, causing all future PSYNC
comparisons to fail permanently — trapping the replica in an infinite full-resync
loop and preventing it from ever catching up.

**Fix:** Added a bounds check immediately after `strtoll()`: offsets that are
negative or exceed `LLONG_MAX / 2` (leaving no room for future offset growth) are
rejected and logged, and the connection falls back to a fresh full resync.

---

### [BUG-08] `dupStringObject` Double-Dup Leaks Intermediate Object
**File:** `src/db.cpp`
**Severity:** High — Memory leak / refcount inconsistency

In `redisDbPersistentData::updateValue()`, when both `old->FExpires()` is true and
`fUpdateMvcc` is true, the code could call `dupStringObject(val)` twice: once for
the expire-update path and once for the MVCC timestamp path. The first duplicate
(refcount = 1, not `OBJ_SHARED_REFCOUNT`) would be overwritten by the second call's
result, leaving the first duplicate unreferenced and leaked. The original shared
object's refcount was also never decremented.

**Fix:** Added a `bool fDuped` flag that tracks whether `val` has already been
duplicated. The MVCC path skips `dupStringObject()` when `fDuped` is already set.

---

## Files Changed

| File | Change |
|---|---|
| `src/networking.cpp` | BUG-01: Reversed lock order in `freeClientAsync` |
| `src/replication.cpp` | BUG-02: Free `bulkreadBuffer` on normal RDB completion path |
| `src/replication.cpp` | BUG-07: Bounds-check `+FULLRESYNC` offset from master |
| `src/bio.cpp` | BUG-03: Cooperative BIO thread exit via `bio_should_exit` flag |
| `src/debug.cpp` | BUG-04: `g_fInCrash` changed to `std::atomic<int>` |
| `src/serverassert.h` | BUG-04: Updated `g_fInCrash` extern and `serverAssert` macro |
| `src/server.h` | BUG-04: Removed duplicate `g_fInCrash` extern, updated `GlobalLocksAcquired` |
| `src/fastlock.cpp` | BUG-04/05: Updated `g_fInCrash` extern; `fInDeadlock` made atomic |
| `src/storage.cpp` | BUG-06: Replace `sfree()` fallback with `serverPanic()` in `pool_free` |
| `src/db.cpp` | BUG-08: `fDuped` guard prevents double `dupStringObject` call |

---

## Validation Recommendations

```bash
# 1. Thread sanitizer build (catches BUG-04, BUG-05 races)
make CFLAGS="-fsanitize=thread -g" LDFLAGS="-fsanitize=thread"
./src/keydb-server --test-memory 100

# 2. Address sanitizer build (catches BUG-02, BUG-06 heap corruption)
make CFLAGS="-fsanitize=address -g" LDFLAGS="-fsanitize=address"

# 3. Replication stress test (validates BUG-02, BUG-07)
./src/keydb-server --port 6379 &
./src/keydb-server --port 6380 --replicaof 127.0.0.1 6379 &
redis-cli -p 6380 info replication  # confirm sync completes

# 4. Concurrent close stress (validates BUG-01 deadlock fix)
redis-benchmark -p 6379 -c 500 -n 100000 -q
```

---

## Upgrade Notes

- No configuration changes required.
- No on-disk format changes.
- Binary-compatible with existing RDB and AOF files.
- Replicas running this build can connect to unpatched masters (and vice versa)
  without protocol issues; the BUG-07 bounds check only rejects malformed offsets.
