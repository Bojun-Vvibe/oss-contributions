# sst/opencode#25014 — test: deflake 'cancel interrupts....' test

- PR: https://github.com/sst/opencode/pull/25014
- HEAD: `f0bd70b`
- Author: rekram1-node
- Files changed: 1 (+10 / -2) — `packages/opencode/test/effect/runner.test.ts`

## Summary

Replaces a 10ms `Effect.sleep` race with a `Deferred<void>` handshake so the
Runner-busy assertion only fires after the inner effect has actually started
running. The pattern is canonical Effect-TS deflake: hand the test a started
signal that the runtime fulfils inside the effect, then `await` it before
sampling state.

## Cited hunks

- `packages/opencode/test/effect/runner.test.ts:115-122` — old code forked
  `runner.ensureRunning(Effect.never...)` then slept 10ms before checking
  `runner.busy === true`. Under load (CI, slow Bun startup, fiber scheduler
  contention) the inner effect could fail to begin within 10ms, leaving
  `busy === false` when the assertion fires.
- `packages/opencode/test/effect/runner.test.ts:118-124` — new code wraps
  `Effect.never` in an outer `Effect.gen` that first does
  `Deferred.succeed(started, undefined)` and then yields `Effect.never`.
  Test then `Deferred.await(started)` before asserting busy state. Once the
  Deferred resolves, the inner effect is provably executing inside the
  Runner.

## Risks

- The `started` deferred is awaited unconditionally — if `ensureRunning`
  itself throws synchronously the test now hangs forever instead of failing
  in 10ms. Consider racing against a finite timeout (e.g. `Effect.timeout`)
  to keep CI bounded if the Runner regresses to never invoking the effect.
- Pattern is local to this one test; other Runner tests at line ~145+ that
  still use `Effect.sleep` would deflake the same way and should follow up.

## Verdict

**merge-as-is**

## Recommendation

Land it; deflake is correct and surgical, follow-up to apply the same
Deferred handshake to sibling tests can be a separate PR.
