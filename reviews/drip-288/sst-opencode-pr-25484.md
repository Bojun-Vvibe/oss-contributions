# sst/opencode PR #25484 — Use Effect helpers in question tests

- Head SHA: `1557f31415b96ac145341c32b94ca90c9d5d155e`
- URL: https://github.com/sst/opencode/pull/25484
- Size: +298 / -321, 1 file (`packages/opencode/test/question/question.test.ts`)
- Verdict: **merge-after-nits**

## What changes

The question service tests are migrated from the legacy
`Instance.provide({ directory, fn: async () => {...} })` pattern + raw
`Promise` choreography to the new shared `testEffect` helper plus
`provideInstance` / `tmpdirScoped` fixtures (drip-287's plumbing).
`ask`/`reply`/`reject` get Effect.fn wrappers (`askEffect`, `replyEffect`,
`rejectEffect`), pending-question waits go through a `waitForPending(n)`
poll (10ms × 100 ticks), and the dangling `ask` is now `Effect.forkScoped`
so the scope automatically interrupts the fiber on test exit.

## What looks good

- `it.instance("...", ..., { git: true })` matches the new fixture-API
  the rest of the suite is moving to. Dropping the explicit
  `await using tmp = await tmpdir(...)` removes a per-test ceremony that
  was easy to forget.
- `Effect.forkScoped` + `Fiber.await` is the right replacement for
  `promise.catch(() => {})` — the old code was leaking unawaited
  rejections into the test runner's unhandled-rejection set.
- `expect((yield* Fiber.await(fiber))._tag).toBe("Failure")` (lines ~75,
  ~95) is a tighter assertion than the previous "just don't throw" — it
  actually verifies the rejection path executed.
- `rejectAll` is now a single `Effect.forEach(..., { discard: true })`
  instead of a `for/await` loop. Same semantics, but composable inside
  other Effects.

## Nits

1. `waitForPending` (around line ~60) is `Effect.sleep("10 millis")` × 100
   = 1s hard cap. That matches what `Bun.sleep` would have done, but the
   error message `"timed out waiting for ${count} pending question
   request(s)"` doesn't include the actual `pending.length` it observed
   on the last poll — adding `(saw ${pending.length})` would make CI
   flakes much easier to diagnose.
2. The test file still imports `AppRuntime` (line 8) and keeps the
   legacy promise-shaped `ask`/`list`/`reply`/`reject` helpers (lines
   ~30-46) "for the dispose/reload tests that exercise instance lifetime
   behavior" (per the PR body). Worth a comment in the file itself
   pointing at *which* downstream tests still need them, otherwise the
   next reviewer will assume they're dead code and try to delete them.
3. `CrossSpawnSpawner.defaultLayer` is merged into the test layer
   (line ~10) but none of the visible tests actually spawn anything —
   they only call `Question.Service`. If `Question.defaultLayer`
   transitively requires `CrossSpawnSpawner` that's fine, but the test
   file should comment why, otherwise it looks copy-pasted.

## Risk

Low — pure test refactor, no production code touched. The behavioral
verification is *strictly stronger* than before (Failure tag check,
forkScoped cleanup) so any regression here would be a real bug surfaced,
not a false positive. The only exposure is test-runtime flake from the
1s `waitForPending` cap if CI is heavily loaded.
