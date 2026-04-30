# sst/opencode #25045 — test: use Effect runtime in runner deadlock case

- **Author:** kitlangton (Kit Langton)
- **SHA:** `586b4c4`
- **State:** OPEN
- **Size:** +44 / -50 across 1 file (`packages/opencode/test/effect/runner.test.ts`)
- **Verdict:** `merge-as-is`

## Summary

Ninth-entry in the running testEffect migration series (drip-194 → drip-202),
this time porting the runner-cancel-deadlock regression in
`packages/opencode/test/effect/runner.test.ts:201-249` from the manual
`Promise<void>` + `setTimeout` + outer `Effect.runPromise(Scope.make())` shape
onto `it.live(...)` + `Effect.gen` with three `Deferred.make<void>()` handles
(`hit`, `hold`, `done`) plus `Effect.forkChild` for the cancellable runs and
`Fiber.await(...).pipe(Effect.timeout("250 millis"))` replacing the
`Promise.race(..., fail(250, "..."))` timeout idiom.

## Reasoning

The bug being locked is the load-bearing one — "cancel must not deadlock when
replacement work starts before interrupted run exits" — and the asserted
ordering (hit → cancel-fork → ensureRunning(b)-fork → busy=true → hold-resolve
→ stop returns success → busy still true → done-resolve → b yields "second" →
busy=false → a is failure exit) is byte-identical to the pre-migration
expectations. Two specific quality wins worth calling out:

1. **The cleanup branch is now `Effect.ensuring(...)`** at the outer
   `Effect.gen` scope instead of a try/finally that could itself throw on the
   `Scope.close(s, Exit.void)` fail-1000ms-timeout race. The new shape lets
   the test fiber's failure surface as the test failure; the pre-migration
   shape would have shadowed the real error with a "runner scope did not
   close" message.

2. **The replacement-work fiber `b` is now `Deferred.await(done).pipe(Effect.as("second"))`**
   inside `runner.ensureRunning(...)` rather than `Effect.promise(() =>
   done.promise)` — eliminates the bridge between Effect's interruption tree
   and the bare-`Promise` continuation that wouldn't honor the cancellation
   protocol if the test were ever interrupted mid-run.

Matches the established testEffect series conventions exactly (the `Layer`
shape, the `Deferred`-rather-than-bare-Promise rule, the
`.pipe(Effect.timeout(...))` failure mode in place of `Promise.race`-with-
sentinel). No behavior surface change. No new public assertions. Migration is
mechanical.
