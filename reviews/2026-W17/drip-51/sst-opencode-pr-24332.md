---
pr: 24332
repo: sst/opencode
sha: 41ecda2d23da52893ab35d6950899cb1eab41ee9
verdict: merge-after-nits
date: 2026-04-26
---

# sst/opencode#24332 — fix: add lock timeout/eviction and fix concurrency issues

- **URL**: https://github.com/sst/opencode/pull/24332
- **Author**: alfredocristofano

## Summary

Three independent fixes bundled:

1. `acp/session.ts` — drop redundant `this.sessions.set(sessionId,
   session)` after mutating the session object in-place.
2. `tool/edit.ts` — add a 60-second eviction timer per file lock so
   the `locks: Map<string, Semaphore>` doesn't grow unbounded over
   long sessions.
3. `util/lock.ts` — add an optional `timeoutMs` parameter (default
   30 000) to the read/write rwlock primitives, racing acquisition
   against a `setTimeout(reject)` and cleaning the waiter out of
   the queue on timeout.

## Reviewable points

- `packages/opencode/src/acp/session.ts:91-110` — the redundant
  `this.sessions.set(sessionId, session)` calls are correctly
  unnecessary because `get(sessionId)` returns the live reference
  and Map values are by-reference. Pure simplification, no
  behavior change. Good.

- `packages/opencode/src/tool/edit.ts:34-58` — the eviction timer
  pattern is right: `scheduleEviction()` after the edit completes,
  cancel-and-clear if the same path is locked again before the
  60s elapses. One subtle issue: the eviction is scheduled in
  the `EditTool.execute` finally branch (line ~184), but if a
  lock is acquired and then the edit *throws*, the eviction is
  never scheduled and the entry leaks until process restart.
  The hot path covers it; the error path doesn't. Suggest moving
  `scheduleEviction()` into a `try/finally` around the inner
  body.

- `packages/opencode/src/util/lock.ts:36-118` — the new timeout
  semantics are correct: the `tryAcquire` closure captures the
  resolve function, gets pushed to the waiter queue, and on
  timeout we splice it out of the queue so `process()` doesn't
  later wake a dead waiter. The closure-identity check via
  `indexOf(tryAcquire)` is the right way to do it.

- One concern at `util/lock.ts:83` — when timeout fires *after*
  `tryAcquire` already ran (race between `Promise.race` resolving
  and the timer), we'll have already incremented `lock.readers`
  but the caller will throw. The reader count then stays
  incremented and the disposable is lost. Net effect: a
  permanently-held read lock against `key`. Suggest checking
  whether the acquisition resolved before discarding it on
  timeout, or wrapping the dispose in the timeout-cleanup path.

- The 30 000 ms default is reasonable for an interactive coding
  agent; long-running grep/format operations under heavy
  concurrency could trip it. A telemetry log on timeout would
  help diagnose.

## Rationale

Two of the three changes are clean. The lock-timeout race
(reader-count leak when timer fires after acquisition) is a real
concern that should be addressed before merge — it's the exact
class of bug the PR is supposed to be fixing.

## What I learned

`Promise.race(acquisition, timeout)` against a queue-based
acquisition is a classic footgun: the timeout doesn't cancel the
work that already started, it only stops awaiting it. The fix is
either (a) make acquisition itself cancellation-aware, or (b)
detect "already acquired" in the timeout branch and route the
disposable somewhere safe (release it immediately).
