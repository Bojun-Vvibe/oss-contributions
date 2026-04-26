# charmbracelet/crush PR #2563 — feat: add undo/redo support for session messages and file restoration

- **PR:** https://github.com/charmbracelet/crush/pull/2563
- **Author:** shahidshabbir-se
- **Head SHA:** `cdb964fe2090`
- **Stats:** +1141 / -100 across 27 files
- **Verdict:** `merge-after-nits`

## What this changes

Adds true session-level undo/redo to crush, with three coordinated pieces: a `revert_message_id` column on the `sessions` table (migration `internal/db/migrations/20260406000000_add_revert_to_sessions.sql`), a backend trio `UndoLastMessage` / `RedoMessage` / `CleanupRevert` exposed both on the in-process `Backend` (`internal/backend/session.go:130-243`) and over the HTTP/proto wire (`internal/client/proto.go:380-417`, `/workspaces/:wsID/sessions/:sID/{undo,redo,cleanup-revert}`), and a TUI affordance via new keybindings + dialog actions (`internal/ui/model/keys.go`, `internal/ui/dialog/{actions,commands}.go`). The file-restoration side is wired through `history.RestoreToTimestamp` and a new `getFileVersionBefore` SQL query so files on disk are walked back to the version that existed *before* the user message we're undoing past.

The state-machine choice is the right one: rather than destructively deleting messages on undo, the design parks a `RevertMessageID` marker on the session and treats messages with `created_at >= marker.created_at` as hidden. Redo just walks the marker forward (or clears it if there's no later user message). `CleanupRevert` is the explicit "commit the undo permanently" action that finally calls `DeleteMessagesAfterTimestamp` and `CleanupAfterTimestamp` — a clean separation between "soft revert with redo available" and "hard prune." The boundary-by-next-message logic in `backendCleanupRevert` (using the *next* user message's timestamp, not the revert message's own, "to avoid same-second timestamp collisions" — `backend/session.go:236-238`) is the kind of detail that only shows up after someone hits the bug; good catch.

## Specific things to fix before merge

**Same-second timestamp collisions deserve more than a comment.** The `RestoreToTimestamp(ctx, sessionID, targetMsg.CreatedAt)` flow at `backend/session.go:139` uses `created_at` as the cutoff key for both messages and file versions. SQLite's default `CURRENT_TIMESTAMP` has 1-second resolution; a fast user pressing send twice in the same second produces two messages with identical `created_at`, and the `>=` / `<` boundary logic will either include both or exclude both depending on which side of the comparison the cutoff falls. The cleanup path papers over this with the next-message trick, but `backendUndo` itself uses the target's own timestamp at line 139 — confirm via test that a same-second double-send doesn't undo *both* messages or leak file versions across the boundary. A monotonically-increasing per-session sequence number (or microsecond-resolution timestamps if SQLite is configured for them) would dodge this entirely.

**The `database/sql.ErrNoRows`/`errors.Is` pattern in `backendRedo` (lines 169-180) has a subtle bug.** The flow checks `err != nil && !errors.Is(err, sql.ErrNoRows)` then `errors.Is(err, sql.ErrNoRows)` separately, which means the "no later user message → restore to head" branch only fires when `next` is the zero value AND `err` is `ErrNoRows`. If `FindNextUserMessage` were to return `nil, nil` for "not found" (a defensible alternative SQL convention), the code would silently call `RestoreToTimestamp(ctx, sessionID, next.CreatedAt)` with `next.CreatedAt` as the zero time, blowing away the entire session's file history. Pin down the contract on `FindNextUserMessage`'s "not found" sentinel in a doc comment and add a test that asserts it.

**`internal/db/db.go` flips `...any` to `...interface{}` in the `DBTX` interface (lines 14-17).** That's a stylistic regression on a Go ≥1.18 codebase — `any` is the canonical alias and the rest of the diff still uses it freely. This looks like the result of regenerating the file with an older `sqlc` version; rerun whatever regenerator the project pins (see `internal/db/sqlc.yaml` or equivalent) so the generated and hand-written files agree on style.

**The TUI surface (`internal/ui/dialog/actions.go`, `internal/ui/dialog/commands.go`, `internal/ui/model/keys.go`, `internal/ui/model/ui.go`) and the workspace plumbing (`internal/workspace/{app,client}_workspace.go`) aren't visible in the first 300 lines of the diff** but together represent the whole "how does a user actually trigger this" surface. Worth confirming that (a) the keybinding doesn't collide with existing global keys, (b) the redo action is greyed out / errors gracefully when no marker is set, (c) there's a visible affordance distinguishing "soft undone, you can redo" from "hard cleaned up, gone forever," and (d) the `mockSessionService.SetRevert/ClearRevert` test stubs at `app/resolve_session_test.go:91-99` actually get exercised by a real test, not just compiled-but-untested.

**The `history.NewService` signature now takes `workingDir`** (`internal/app/app.go:82`, `internal/agent/common_test.go:121`). Confirm there are no third-party embedders of crush that construct `history.NewService` directly — if there are, this is a breaking API change worth a CHANGELOG note.

## Bottom line

Substantial, well-shaped feature with the right state-machine model (soft-marker + explicit cleanup) and the right separation of concerns across DB / backend / TUI. Pin down the timestamp-collision and `FindNextUserMessage`-sentinel contracts with tests, fix the `any` → `interface{}` regression, and confirm the TUI affordances. After that this should land.
