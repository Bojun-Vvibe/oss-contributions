# sst/opencode #24154 — feat: add unarchive/restore for archived sessions

**Link:** https://github.com/sst/opencode/pull/24154
**Tag:** symmetry-fix, ACL-gap

## What it does

Closes #24153. Closes the asymmetry where session archive is one-way:
the `UpdatedTime` schema already accepts `NullOr(Number)` server-side,
but the PATCH `/session/:id` validator and `setArchived` projector
didn't pass `null` through, and no UI surface called for it. This PR
plumbs the missing pieces:

- **Backend:** PATCH validator now accepts `null` for `time.archived`;
  `setArchived` forwards `null` so SQLite gets `time_archived = NULL`.
- **Web:** `unarchiveSession()` calls `update({ time: { archived: null
  } })` then navigates back to the restored session; command palette
  gains "Unarchive session"; session dropdown swaps Archive↔Unarchive
  via `info()?.time?.archived`; file picker labels archived rows
  `(archived)`.
- **TUI:** `ctrl+a` on a session row toggles archive/unarchive (single
  bind, branches on `session.time?.archived`); list rows get
  `[archived]` prefix.
- **Event reducer:** increments `sessionTotal` when an unarchived
  session is reinserted into the store (top-level only, gated on
  `!info.parentID`).

## What it gets right

- Reuses the existing `UpdatedTime` schema's `NullOr(Number)` rather
  than inventing a new "unarchive" verb. The semantic clearing-via-null
  is symmetric with how `archive` was already encoded as
  `archived: timestamp`. Keeps the wire surface minimal.
- Single TUI keybind for both directions (branch on current state) is
  the right ergonomic — operators don't have to remember two binds.
- Web dropdown uses `<Show fallback={…}>` to render exactly one of
  Archive/Unarchive, never both — no dead code path possible.
- The `sessionTotal` increment is correctly scoped to top-level sessions
  (`!info.parentID`), so reinserted child threads don't double-count.

## Concerns / risks

- **No ACL check on unarchive.** Archive was authorize-as-owner; the
  new `unarchive` path inherits whatever the PATCH endpoint already
  enforces, but the PR doesn't exercise that path in tests. Worth a
  regression test that a non-owner can't `update({ time: { archived:
  null } })` to resurrect somebody else's archived session.
- **Event reducer increment is unconditional on insert.** If the
  upstream broadcast retransmits an `info.updated` event for an already
  re-inserted unarchived session, `sessionTotal` ticks up again. The
  `reconcile` call dedupes the row but the counter mutation runs first.
  A guard like `if (!previousList.some(s => s.id === info.id))` would
  make it idempotent.
- The TUI keybind is `ctrl+a` with no confirmation. Archive is
  reversible now (good), but a fat-fingered `ctrl+a` on a wrong row
  will silently move it to the archive bucket. Either show a 2s
  toast with an undo, or require shift modifier.
- The file picker's `(archived)` label uses `text-text-weak` — fine on
  light theme; on dark themes with low-contrast weak-text colors this
  may be sub-AA. Worth a contrast spot-check.

## Suggestion

Add a regression test asserting `update({ time: { archived: null } })`
respects the same actor-permission check as the archive direction. The
unarchive path is a new write surface and deserves explicit ACL
coverage, not just "the PATCH route already handles it".
