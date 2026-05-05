# sst/opencode#25798 — fix(session): cancel subtask child sessions

- Head SHA: `ad9eefd48719d80e6c6a2b80764ee3e5f2994035`
- Author: kitlangton
- Verdict: **merge-after-nits**

## Summary

Propagates Task tool cancellation into child sessions. Changes
`TaskPromptOps.cancel` from a fire-and-forget `void` returner to an
awaitable `Effect.Effect<void>`, then uses `Effect.acquireUseRelease` so
that release-time cleanup actually waits for the child cancellation when
the parent fiber was interrupted. Adds regression coverage for both the
slash-command subtask path and a direct Task abort signal.

## Specific references

- `packages/opencode/src/session/prompt.ts:122` — `cancel` is now the raw
  `cancel(sessionID)` Effect rather than `run.fork(cancel(sessionID))`.
  This is the load-bearing change: the `EffectBridge` runner is no longer
  needed at the layer level because the caller controls forking now.
- `packages/opencode/src/tool/task.ts:122` — `TaskPromptOps.cancel`
  signature flips from `void` to `Effect.Effect<void>`. Any out-of-tree
  consumer of this interface will break at compile time, which is the
  right behavior for a contract change.
- `packages/opencode/src/tool/task.ts:166-178` — release callback now
  receives `(_, exit)` and only awaits `cancel` when
  `Exit.hasInterrupts(exit)`. The `Effect.ensuring` wrapper guarantees
  the abort listener is removed even if the cancel Effect itself fails.
- New tests under `packages/opencode/test/` exercise both the
  slash-command subtask interrupt and direct `ctx.abort.abort()` paths.

## Reasoning

The bug class here is real — a fire-and-forget cancel via `run.fork`
returns immediately, so the parent's release callback completes before
the child session has actually torn down. On rapid abort/restart cycles
that left orphan child sessions running and emitting events into a
disposed parent. Awaiting the cancel only on `Exit.hasInterrupts` is the
correct narrowing: normal completion shouldn't pay the cancel cost.

Nit: the `runCancel` `EffectBridge` instance built at
`packages/opencode/src/tool/task.ts:122` is only used by `onAbort`'s
synchronous handler — the release path now awaits the same `cancel`
Effect directly. Worth a comment explaining why two different invocation
sites (sync DOM event handler vs Effect release) need both shapes; a
future refactor might want a single helper.

Nit: the regression-origin commit hash in the PR body is a useful
forensic breadcrumb but doesn't appear in any code comment near
`task.ts:122`. Adding a one-liner like `// see <SHA>: defineEffect →
define collapsed the awaitable cancel path` would help the next person.
