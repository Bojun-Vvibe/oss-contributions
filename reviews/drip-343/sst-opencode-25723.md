# sst/opencode #25723 — fix(worktree): fork workspace worktree boot

- PR: https://github.com/sst/opencode/pull/25723
- Head SHA: `30d90204b2b1c4cd52bffb211312306680929425`
- Diff size: +111 / -6 across 2 files (1 src + 1 new test)

## Summary

Fixes a regression where the experimental workspaces HTTP API
worktree-create endpoint would *block* on the bootstrap step, causing
the request to hang indefinitely (or until the start command finished).
The fix moves the `boot(...)` call into a forked Effect inside the
shared `createFromInfo` helper, so the HTTP handler returns as soon as
the worktree is set up. Adds a regression test that asserts both the
direct and workspace-routed worktree endpoints return promptly without
waiting for boot.

## Citations

- `packages/opencode/src/worktree/index.ts:291-302` — the meaningful
  change. Previously, only `Worktree.create` forked the boot step;
  `Worktree.createFromInfo` ran it inline. The HTTP-API path goes
  through `createFromInfo`, which is why the endpoint hung. After
  this PR, both code paths share the same forked-boot pattern by
  consolidating into `createFromInfo`. The deletion in
  `Worktree.create` (the same `setup + forked boot` block) is
  recovered by calling `createFromInfo` from `create`.
- `worktree/index.ts:294-297` — the fork uses
  `Effect.catchCause((cause) => Effect.sync(() => log.error(...)))`
  before `Effect.forkIn(scope)`. This means a boot failure will be
  logged but otherwise *swallowed* — there is no telemetry event,
  no failure surfaced to the API caller, no retry. For an
  experimental endpoint this is acceptable; for a stable endpoint
  it would not be. Worth a follow-up to surface the error via an
  event the UI can subscribe to.
- `test/server/worktree-endpoint-repro.test.ts` (new, +106 LOC) —
  good repro coverage. Sets up flags
  `OPENCODE_EXPERIMENTAL_HTTPAPI` / `OPENCODE_EXPERIMENTAL_WORKSPACES`
  to true, calls the create endpoint with a 5s `withTimeout`, and
  asserts a 200 response. The 5s timeout is generous enough to not
  flake but tight enough to actually catch the hang. Good test.
- The test sets and restores `Flag.OPENCODE_EXPERIMENTAL_*` in
  `afterEach` — but the assignment to `Flag.X` happens inside the
  `app()` factory which is called *during* the test, not in
  `beforeEach`. If a test throws before `using server = app()` runs,
  the original values are still restored — fine. But if multiple
  tests in this file run in parallel (Bun test runner default), the
  global `Flag` mutation races. The `afterEach` reset assumes
  serial execution. Worth confirming this file is configured for
  serial test execution or refactoring to a per-test scope.

## Risk

Low for the experimental endpoint. The src change is symmetric
(both creation paths now use the same forked-boot pattern that
`Worktree.create` already had). Test changes are additive.

## Verdict

`merge-after-nits` — the fix is correct and well-tested. The
swallowed-error pattern in the forked boot needs a follow-up
(event surfacing) and the test file's global flag mutation
should be made parallel-safe before this lands in a non-experimental
context.
