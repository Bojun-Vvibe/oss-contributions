# sst/opencode PR #25048 — test: use Effect test helper for run-service

- PR: https://github.com/sst/opencode/pull/25048
- Head SHA: `97dca9abd4454da0e44c82f2e83f5d719a6d3da2`
- Files touched: 1 (`packages/opencode/test/effect/run-service.test.ts` +39 / -36, net +3 lines).

## Specific citations

- The migration swaps `import { expect, test } from "bun:test"` → `import { expect } from "bun:test"` plus a new `import { it } from "../lib/effect"` at `:1-4`, matching the established testEffect-migration shape from drip-194 #25053, drip-195 #25052, drip-196 #25050/#25051, drip-198 #25049.
- `test("makeRuntime shares dependent layers through the shared memo map", async () => { ... })` at `:6` is replaced with `it.live("makeRuntime shares dependent layers through the shared memo map", () => Effect.gen(function* () { ... }))` at `:6-49`. The body is wrapped in `Effect.gen` and the two terminal `await runOne(...)` / `await runTwo(...)` calls are converted to `yield* Effect.promise(() => runOne(...))` / `yield* Effect.promise(() => runTwo(...))` at `:46-47`.
- The `let n = 0` counter and the three `Layer.effect` constructions for `Shared`, `One`, `Two` are preserved verbatim — the migration is purely the harness wrapper, not a rewrite of the test logic. The final three assertions (`.toBe(1)`, `.toBe(1)`, and `expect(n).toBe(1)` at `:48`) are the unchanged invariants: both `One` and `Two` see the same `Shared.id`, and `Shared` is constructed exactly once (the memo-map property the test is named after).

## Verdict: merge-as-is

## Rationale

This is a mechanical migration onto the established testEffect surface, matching the running series of `it.live`-based test migrations that have been landing across the past five drips. The pattern is well-understood at this point: replace `test(name, async () => ...)` with `it.live(name, () => Effect.gen(...))`, wrap the assertions in `yield* Effect.promise(...)` for any `runPromise`-style call site, and let the shared `lib/effect` harness handle scope/finalizer/runtime tracking instead of bun's bare async harness. The diff is +3 net lines, the assertion semantics are byte-identical, and the headline invariant (the `Shared` layer's memo-map property — `n === 1` after both `runOne` and `runTwo` have read it) is preserved verbatim.

The `it.live` choice (vs `it.effect`) is correct here because `makeRuntime` constructs and shares real `Effect.Runtime` instances and the test exercises the actual `runPromise` path twice — using `it.effect` (which runs against the test runtime/`TestClock`) would change what's being tested. The two `Effect.promise(() => runOne(...))` wrappers at `:46-47` are the right shape for bridging external Promise-returning APIs back into the Effect generator without introducing a `Effect.tryPromise` failure channel that the test doesn't care about.

There's no behavior change, no test-logic change, no fixture change. The only conceivable nit would be the `.live` choice if the test were intended to run against `TestClock`, but `makeRuntime` is a runtime-construction primitive that has no time-dependent semantics here, so `.live` is the right answer. Net: merge-as-is, this is the seventh entry in the series and the pattern has stabilized.

## What I learned

The cumulative testEffect migration across this codebase is a good example of a "small mechanical change × N files" refactor where the per-PR review fatigue is real but the policy itself is correct: lifting tests onto a shared Effect-aware harness lets the harness own scope/finalizer/runtime semantics and stops every individual test from re-implementing them in slightly different ways with bare bun async. Reviewing each PR in isolation looks like noise; reviewing the series as a whole, the value is the elimination of an entire bug class (test-leaked finalizers, runtime-instance accumulation across tests, scope handles outliving the test that opened them). The right move when a series like this lands is to spot-check that the migration didn't accidentally change `it.live` to `it.effect` (which would silently swap real-runtime behavior for fake-clock behavior) — confirmed `.live` here is the correct choice.
