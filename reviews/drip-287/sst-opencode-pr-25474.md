# sst/opencode PR #25474 — refactor(cli): convert stats command to effectCmd

- Head SHA: `fc7643be726d67ab6557d83a4b7b43b110354d99`
- URL: https://github.com/sst/opencode/pull/25474
- Size: +33 / -28, 1 file
- Verdict: **merge-after-nits**

## What changes

`packages/opencode/src/cli/cmd/stats.ts` is converted from `cmd(...)` +
`bootstrap(...)` to `effectCmd`. The previous `getCurrentProject()` helper
that read `Instance.project` from ambient context is removed; the project
is now passed in explicitly to `aggregateSessionStats`.

## What looks good

- Removing `getCurrentProject()` and threading `currentProject` as an
  explicit parameter makes `aggregateSessionStats` testable in isolation —
  the function previously had a hidden dependency on the `Instance`
  global.
- Auto-dispose via `Effect.ensuring(store.dispose(ctx))` is consistent
  with the rest of the effectCmd migration sequence (#25471, #25473,
  #25481, #25485).

## Nits

1. The new error path `throw new Error("currentProject required when projectFilter is empty string")`
   in `aggregateSessionStats` (around the modified line ~125) regresses
   the API: callers that previously could pass `projectFilter=""` and rely
   on ambient `Instance.project` resolution will now get a runtime error
   if they forget to pass `currentProject`. Since `aggregateSessionStats`
   is **exported**, this is a breaking change for any in-tree consumer
   (and potentially plugins). Either:
   - Make `currentProject` non-optional at the type level so the breakage
     is a compile error, not a runtime error, OR
   - Keep the parameter optional and fall back to a clearer
     `BadRequestError`/typed error rather than a bare `throw new Error`.
2. `if (!ctx) return` silently no-ops when `InstanceRef` is absent — same
   issue as #25485. A short stderr message would help users understand
   why `opencode stats` produced no output.
3. The `run` arrow function signature uses inline `{ days?: number; tools?: number; models?: unknown; ... }`
   — extracting that as a named type (or inferring it from yargs builder)
   would prevent drift if the builder gains options later.

## Risk

Low for CLI users (behavior identical when invoked via the command).
**Medium** for in-tree code that imports `aggregateSessionStats` —
worth a one-line CHANGELOG note about the new required arg.
