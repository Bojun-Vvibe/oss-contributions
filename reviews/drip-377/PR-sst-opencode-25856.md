# sst/opencode #25856 — feat(todo): auto-cleanup stale todos + /clear-tasks and /清除任务 commands

- Head SHA: `c1769f40e1d3a139c4997a535033c49236a01e2c`
- Diff: +25 / -2 across 4 files (i18n, command index, session/todo, todowrite tool prompt)

## Findings

1. The load-bearing behavioural change is the read-side filter at `packages/opencode/src/session/todo.ts:73`: `rows.filter((row) => row.status === "pending" || row.status === "in_progress").map(...)`. Completed/cancelled rows remain in the SQLite `TodoTable` (the `INSERT` path is unchanged) but are never re-surfaced on session load — so the change is non-destructive at the storage layer (a future migration could re-expose history) while being effective at the UI/agent layer. Good design.
2. The slash commands at `packages/opencode/src/command/index.ts:104-120` register two parallel commands (`clear-tasks` and `清除任务`) with identical templates: `"Call todowrite with an empty todos array [] to clear all remaining tasks."` This delegates the actual mutation to the model rather than calling the DB directly — consistent with how `INIT` and `REVIEW` commands work, and means the action shows up in the transcript as a tool call (auditable). Concern: an empty `todos: []` write in the existing `todowrite` tool may interpret "no todos" as "no-op" rather than "clear all"; worth confirming the tool actually clears (not no-ops) on empty input before merge — if it no-ops, the slash commands silently fail.
3. The `command.category.tasks` translation key is added to `packages/app/src/i18n/zh.ts:23` but the new commands at `command/index.ts:104-120` don't set a `category` field, so this key is currently unreferenced. Either set `category: "tasks"` on both command registrations or drop the unused i18n key.
4. Tool-prompt edit at `packages/opencode/src/tool/todowrite.txt:1` adds the sentence `"Completed/cancelled tasks are auto-removed for your current coding session."` — this correctly informs the model of the new semantics so it doesn't try to "preserve completed items for context", but the phrasing "auto-removed" is slightly misleading because rows persist in the DB. Suggest `"Completed/cancelled tasks are hidden from session reload"` for accuracy.

## Verdict

**Verdict:** merge-after-nits
