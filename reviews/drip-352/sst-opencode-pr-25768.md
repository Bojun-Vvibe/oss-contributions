# sst/opencode #25768 — Jlongster/warp 2

- URL: https://github.com/sst/opencode/pull/25768
- Head SHA: `098258817ae41e8a0cde56c6ee172ef4c80c91ee`
- Author: jlongster
- Size: +4036 / -1089 across 25 files (sync, control-plane, TUI workspace dialogs, SDK gen)

## Comments

1. `packages/opencode/migration/20260504145000_add_sync_owner/migration.sql:1` — Single ALTER adds `owner_id` column without NOT NULL / default, which is correct for a backfill migration but the file ends without a trailing newline (`\ No newline at end of file`). Worth fixing for POSIX-tool friendliness; not blocking.
2. `packages/opencode/src/sync/event.sql.ts` / `src/sync/index.ts` — The new `owner_id` column needs every existing INSERT path covered. Spot-check that the `event_sequence` insert in `src/sync/index.ts` always populates `owner_id` (or explicitly NULLs it for legacy rows) so reads can rely on it later for tenant isolation.
3. `packages/opencode/src/server/routes/instance/httpapi/handlers/workspace.ts` + `groups/workspace.ts` — New workspace-scoped HTTP handlers should have an authz check tied to `account_state.active_account_id` before mutating; please confirm the route group middleware enforces it (the diff is large; reviewer needs to verify, not just trust).
4. `packages/opencode/src/cli/cmd/tui/component/dialog-workspace-create.tsx` — New TUI dialog: confirm focus restoration on `Escape` returns to the prior pane; new dialogs in this codebase have historically leaked focus to the prompt (see #25667 thread).
5. `packages/sdk/openapi.json` and `packages/sdk/js/src/v2/gen/*` — These are regenerated artefacts. PR title `Jlongster/warp 2` is non-descriptive; please rename to a conventional summary (`feat(sync): scope event sequence to owner workspace`) so the changelog is usable.
6. `packages/opencode/test/sync/index.test.ts` — Good that a sync test exists; please ensure it covers both the migration-path (existing rows with NULL `owner_id`) and the fresh-write path.

## Verdict

`needs-discussion`

## Reasoning

This is a 4k-line cross-cutting change touching schema, sync, control-plane HTTP, TUI dialogs, and the generated SDK — all behind an unreviewable PR title (`Jlongster/warp 2`). Even if each individual hunk is sound, a change of this shape needs (a) a real PR description explaining the workspace-owner model, (b) confirmation that the new HTTP handlers are gated behind the existing auth layer, and (c) a migration-compat statement for clients that don't yet know about `owner_id`. Recommending a rebase into a stack of smaller PRs (migration → sync writer → HTTP routes → TUI) before merge.
