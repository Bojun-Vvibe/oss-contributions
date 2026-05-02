# sst/opencode PR #25483 — refactor(cli): convert session subcommands to effectCmd

- Head SHA: `72882a4b5b74d73f7a1c43253df181df68b52061`
- URL: https://github.com/sst/opencode/pull/25483
- Size: +49 / -53, 1 file (`packages/opencode/src/cli/cmd/session.ts`)
- Verdict: **merge-after-nits**

## What changes

`SessionDeleteCommand` and `SessionListCommand` are migrated from
`cmd(...)` + `bootstrap(process.cwd(), async () => ...)` to `effectCmd`
+ `Effect.ensuring(store.dispose(ctx))`. `Session.Service` is yielded
directly (no more `AppRuntime.runPromise(Session.Service.use(...))`
wrapping). The `try/catch` around `Session.get(sessionID)` becomes
`Effect.catchCause(() => fail("Session not found: ..."))` — the PR body
correctly notes that `Session.get` surfaces `NotFoundError` as a defect,
not a typed E channel, so `catchCause` (not `catchAll`) is required.
Pager invocation stays inside `Effect.promise(async () => ...)`. The
parent `SessionCommand` group dispatcher stays on `cmd()` since it has
no body.

## What looks good

- The behavioral note in the PR body about `session delete <bad-id>` is
  the strict-improvement case: legacy `process.exit(1)` (line ~67 of
  the old file) bypassed `bootstrap`'s finally, so instance dispose was
  skipped. New `fail(...)` flows through `Effect.ensuring(store.dispose(ctx))`
  before the runtime exits with code 1 — same UX, cleaner shutdown.
- `Effect.catchCause(() => fail(...))` is the right primitive choice.
  Using `catchAll` would have silently missed the defect path and the
  bad-id case would have crashed with the raw `NotFoundError` cause.
- Pager spawn correctly stays imperative inside `Effect.promise` —
  trying to lift `Process.spawn` + stdio piping into Effect would have
  obscured the lifecycle for no win.
- The `SessionListCommand` short-circuit `if (sessions.length === 0)
  return` (line ~96) is preserved verbatim and still flows through
  `ensuring`, so dispose runs on the empty-list path too.

## Nits

1. `if (!ctx) return` (lines 68 and 95 of the new file) silently
   no-ops when there is no `InstanceRef`. Same observation as drip-287
   #25485: legacy `bootstrap(process.cwd(), ...)` would proceed using
   `cwd`, so a user running `session list` outside any project gets
   *nothing* now where they previously got "no sessions" or actual
   sessions from a parent dir. Worth at least a stderr line.
2. The pager block (lines ~104-120 of the new file) wraps the *entire*
   spawn + write + `await proc.exited` in a single `Effect.promise`.
   That's correct, but if `proc.stdin` is unexpectedly null mid-flight
   the `console.log(output)` fallback fires inside the Effect and the
   promise resolves — fine, but the legacy version had the same shape,
   so no regression. Just call out in the PR that the fallback is
   intentional.
3. The PR body says `db.ts` was considered next but skipped because
   "none of its subcommands use bootstrap or any services (just spawns
   sqlite3, runs JSON migration)". Good call — adding instance load
   there would be wasted work. Worth a one-line `// db.ts intentionally
   stays on cmd() — no instance context required` comment somewhere
   discoverable so the next migrator doesn't second-guess.

## Risk

Low — pure CLI plumbing. No wire-format change, no behavior change for
the happy paths, and the bad-id path is *strictly* more correct
(dispose now runs before exit). 137 CLI tests pass per PR body.
