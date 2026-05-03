# sst/opencode PR #25359 — workspace warp / steal / erase rework

- **Repo:** sst/opencode
- **PR:** #25359
- **Head SHA:** `7089f72e761c76abcf42dd68c61abbba7d94ff7f`
- **Author:** jlongster
- **Title:** warp
- **Diff size:** +967 / -941 across 18 files
- **Drip:** drip-295

## Files changed (substantive subset)

- `packages/opencode/src/cli/cmd/tui/component/dialog-workspace-create.tsx` (+130/-109) — `restoreWorkspaceSession` is renamed to `warpWorkspaceSession`, calls the new SDK method `experimental.workspace.warp(...)`, returns `Promise<boolean>`, gains `showSuccessToast` opt-out. `DialogWorkspaceCreate` is replaced by `DialogWorkspaceSelect` exposing existing-workspace selection.
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` (+298/-112) — adds `warpSession(selection)` flow at line ~581 wired to two `<DialogWorkspaceSelect>` invocations.
- `packages/opencode/src/cli/cmd/tui/component/dialog-session-list.tsx` (+46/-66) — drops the `ctrl+w "new workspace"` shortcut and the inline `WorkspaceStatus` rendering; delegates to the new `WorkspaceLabel` component.
- `packages/opencode/src/cli/cmd/tui/component/workspace-label.tsx` (+19/-0, new) — extracted footer renderer.
- `packages/opencode/src/control-plane/workspace.ts` (+142/-126) — workspace lifecycle changes (handler renames; `sessionRestore` removed at the route level — see test deletion below).
- `packages/opencode/src/server/routes/instance/sync.ts` (+83/-1) — adds two new POST endpoints under the sync router: `/erase` (deletes from `EventTable` and `EventSequenceTable` for a session) and `/steal` (writes a `Session.Event.Updated` event reassigning `workspaceID` via `WorkspaceContext.workspaceID`).
- `packages/opencode/src/server/routes/instance/httpapi/{groups,handlers}/workspace.ts` (-18, -17) — removes the prior `sessionRestore` httpapi binding entirely.
- `packages/opencode/test/control-plane/workspace.test.ts` (+4/-366) — large net-deletion of the workspace control-plane test suite.
- `packages/opencode/test/server/httpapi-workspace.test.ts` (-12) — deletes httpapi workspace test cases.
- `packages/sdk/js/src/v2/gen/{sdk,types}.gen.ts` (+159/-32) — generated SDK delta; mostly the new `warp` method.

## Specific observations

- `sync.ts:111-148` (the new `/erase` handler) — runs `Database.transaction((tx) => tx.delete(EventTable).where(eq(...)).run(); tx.delete(EventSequenceTable)...)`. Two issues: (1) the transaction callback is synchronous and the request handler is `async`, so error propagation from inside the txn relies on drizzle-better-sqlite throwing — confirm that pattern is consistent with the rest of `sync.ts`. (2) There's no authorization check — any caller with a `sessionID` can wipe its event log. If this is the intended trust model (loopback HTTP only), say so in the PR description; otherwise add at minimum a session-belongs-to-this-instance check.
- `sync.ts:149-185` (the new `/steal` handler) — `if (!workspaceID) throw new Error("Cannot steal session without workspace context")` returns a 500, not a 4xx. The endpoint is documented to return `400` in the `errors(400)` list, so wire this through `HTTPException` / the codebase's typed error helper.
- `sync.ts:178-184` — `SyncEvent.run(Session.Event.Updated, { sessionID, info: { workspaceID } })` — the `info` object only carries `workspaceID`. If `Session.Event.Updated` is consumed elsewhere with the assumption that `info` is the full session info object, the partial payload will overwrite other fields with `undefined` on rehydrate. Worth grepping `Session.Event.Updated` consumers; if they merge against the prior state this is fine, if they replace, this is silent data loss.
- `dialog-workspace-create.tsx:267-310` (the renamed `warpWorkspaceSession`) — the `.catch(() => undefined)` on the SDK call swallows the underlying error, then `result?.error` is reported via `errorMessage`. If the catch fires, `result` is `undefined` and the user sees "no response" with no diagnostic. Either log the swallowed error or let the SDK error surface.
- `dialog-workspace-create.tsx:42-50` — return type changed from `void` to `Promise<boolean>`. Good for the new `prompt/index.tsx:581` `warpSession` consumer that awaits a confirmation. But the old `done?.()` callback contract is partially preserved (`if (input.done) return true`) — clarify in a doc comment which signal callers should use (return value vs `done` callback), otherwise integrators will use both.
- `test/control-plane/workspace.test.ts` (-366 lines net) — removing 366 lines of workspace tests for what looks like a behavioral migration is a yellow flag. The diff is a near-total delete plus a small rewrite. Confirm this is actually replacement coverage and not a test-suite haircut. If the prior tests asserted `sessionRestore` semantics that are now provided by `/erase` + `/steal`, the new endpoints need symmetric tests in `test/server/sync.test.ts` (not present in the diff).
- `test/server/httpapi-workspace.test.ts` (-12) — same concern; the deleted httpapi handler had test coverage that no longer exists for the replacement sync routes.
- `dialog-session-list.tsx:239` — the `ctrl+w "new workspace"` keybind is removed without replacement in the keymap. If the new flow is "select-or-create", the equivalent shortcut should still exist; otherwise users behind the experimental flag lose a keybinding.
- The PR title is just "warp", which makes the intent legible only to people in the room. The summary of "rename `restoreWorkspaceSession` → `warpWorkspaceSession`, replace the workspace-create dialog with a select dialog, and route session-reassignment through two new sync endpoints" deserves to be in the PR body before merge.

## Verdict: `request-changes`

Three blockers: (1) the deleted workspace + httpapi tests (-378 lines) need replacement coverage on the new `/erase` + `/steal` sync endpoints — both are destructive and currently untested; (2) `/erase` needs an authorization story or an explicit "loopback-only, intentional" note in the code; (3) the `/steal` `Session.Event.Updated` payload only carries `workspaceID` — confirm consumers merge rather than replace before this lands. The TUI surface is fine; the server surface is what's underbaked.
