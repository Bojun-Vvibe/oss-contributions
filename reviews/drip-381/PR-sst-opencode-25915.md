# sst/opencode PR #25915 — fix(tui): filter only connected workspaces in dialog

- URL: https://github.com/anomalyco/opencode/pull/25915
- Head SHA: `a221a11a2a99836ce63b5ab8691e1999d31b0166`
- Size: +80 / -17

## Summary

Refactors the recent-workspace selection logic in the TUI workspace-create dialog so it (a) is unit-testable, (b) only surfaces *connected* workspaces, and (c) hides the "View all workspaces" footer entry when the recent list already covers the full set. Extracts a pure helper `recentConnectedWorkspaces({ sessions, get, status, limit })` at `dialog-workspace-create.tsx:36-52` and adds a 38-line bun:test exercising the dedup + missing + status-filter axes.

## Specific findings

- `dialog-workspace-create.tsx:36-52` — new exported `recentConnectedWorkspaces<WorkspaceInfo>` helper. Sorts sessions descending by `time.updated`, lifts each `workspaceID` to the `WorkspaceInfo` via the injected `get`, gates on `status(workspace.id) === "connected"`, dedups on `id` via `findIndex` (O(N²) but N≤recent-history, negligible), and slices to `limit ?? 3`. Returns `{ recent, hasMore }` where `hasMore = recent.length < workspaces.length` — i.e., true iff the cap actually trimmed something.
- `dialog-workspace-create.tsx:146-150` — call site cleanly replaces the inlined-then-fragile chain that previously did `flatMap(... ? [workspaceID] : [])` then `.flatMap(...)` then `.slice(0,3)` with a single helper invocation.
- `dialog-workspace-create.tsx:175-184` — `View all workspaces` entry is now wrapped in `...(hasMore ? [{...}] : [])`. **Subtle bug risk:** `hasMore = recent.length < workspaces.length` evaluates **after** the limit is applied, so when `workspaces.length === 0` (no connected workspaces in any session) `hasMore` is `false` and the footer is hidden — meaning a brand-new user with zero session history cannot reach the full workspace list from this dialog. May be intentional (the `list.map((adapter) => ...)` adapter rows still populate the dialog), but worth a one-line code comment confirming the design.
- `dialog-workspace-create.tsx:97` — drive-by simplification `id: input.workspaceID ?? undefined` → `id: input.workspaceID` because the param type is already `string | undefined`. Clean.
- `dialog-workspace-create.test.ts:1-38` — new test file. One test covers: (a) session with no `workspaceID` is skipped, (b) `wrk_b` (disconnected) and `wrk_c` (error) are filtered, (c) `wrk_missing` (not in `get` result) is dropped, (d) duplicate `wrk_a` from two sessions collapses to one, (e) order respects most-recent-first per session timestamp. Final assertion `["wrk_a", "wrk_d", "wrk_e"]` is correct given the input.
- `packages/sdk/openapi.json:8408-8418` — auto-generated change widening `experimental.workspace.warp` request `id` from `string` to `anyOf: [string, null]`. This is a real wire-protocol shape change for SDK consumers — backward-compatible for clients sending strings, but downstream typed bindings will regenerate to `string | null`. Worth a release-note callout since it touches the public OpenAPI spec.

## Notes

- Test file is missing a "limit honored when more than 3 connected workspaces exist" axis — easy add.
- Test file is missing a trailing newline (pattern persists from earlier opencode PRs in this drip series).

## Verdict

`merge-after-nits`
