# sst/opencode #25021 — test: deflake runner cancel test

- **URL:** https://github.com/sst/opencode/pull/25021
- **Head SHA:** `3c9ca62d46af7ab8ebafc28c49726235142c07c6`
- **Files:** `packages/opencode/test/effect/runner.test.ts` (+10/-2)
- **Verdict:** `merge-as-is`

## What changed

Replaces a 10ms `Effect.sleep` race with a `Deferred<void>` handshake at `runner.test.ts:118-124`. The `ensureRunning` payload now wraps `Effect.never` in an `Effect.gen` that signals `Deferred.succeed(started, void 0)` *before* parking, and the test awaits `Deferred.await(started)` instead of sleeping before asserting `runner.busy === true` and `runner.state._tag === "Running"`.

## Why it's right

- The pre-image relied on `Effect.sleep("10 millis")` to give the forked fiber time to enter the inner effect — classic race that fails on slow CI runners or when the runtime scheduler is contended. The new shape is provably correct: the assertion fires only after the inner effect has demonstrably begun executing.
- This is the canonical Effect-TS deflake pattern (signal-via-Deferred, not sleep). The signal happens *before* `Effect.never`, so by the time the test thread resumes, `runner.ensureRunning` has already transitioned state to `Running`.
- Surface area is tiny — single test, no production code touched, no behavioral change to `Runner`.

## Nits / not blockers

- None worth holding the PR for. A future drive-by could lift the `started`-Deferred + signal-then-park pattern into a tiny test helper (`yieldAndPark(deferred)`) since this idiom recurs in Effect-TS test suites, but that's a follow-up not a blocker here.

## Risk

Negligible. Test-only change. The previous flake (sleep-then-assert on a forked fiber's state) is exactly the bug class this fixes; if anything regresses it would be a tighter assertion failing more loudly, not a false-pass.
