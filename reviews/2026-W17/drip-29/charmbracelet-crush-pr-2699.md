# charmbracelet/crush#2699 — fix(lsp): enforce workspace boundary for workspace edits

- PR: https://github.com/charmbracelet/crush/pull/2699
- Author: iceymoss
- +202 / -16
- Head SHA: `7db950d1d6e2aed78688c90c6667a180f87d963d`

## Summary

Adds workspace-root validation to every filesystem-mutating path under the
LSP `workspace/applyEdit` flow. Before this change, a malicious or buggy
LSP server could return a `WorkspaceEdit` whose URIs pointed anywhere on
disk and `crush` would happily apply text edits, file creates, recursive
deletes, and renames against those out-of-workspace paths. The fix
threads `workspaceRoot` (sourced from `Client.cwd`) down through
`ApplyWorkspaceEdit` → `applyTextEdits` / `applyDocumentChange`, and
introduces `validateWorkspacePath` which rejects any URI whose resolved
absolute path is not under the workspace root. A separate helper
`normalizeURIPath` handles a Windows-specific quirk where LSP
`DocumentURI.Path()` returns `/C:/...` or `\C:\...` and `filepath.Abs`
would otherwise produce nonsense like `D:\C:\...`.

## Specific findings

- `internal/lsp/util/edit.go:32` (SHA
  `7db950d1d6e2aed78688c90c6667a180f87d963d`) —
  `validateWorkspacePath` resolves both the URI path and the workspace
  root with `filepath.Abs`, then checks `fsext.HasPrefix(absPath, root)`.
  Using a project-internal `HasPrefix` (rather than naive `strings.HasPrefix`)
  is the right call — assuming `fsext.HasPrefix` correctly handles the
  `/foo` vs `/foobar` boundary case (i.e., requires a trailing separator
  match, not a substring match). Worth a one-line spot-check on the
  `fsext` implementation.
- `internal/lsp/util/edit.go:14` — `normalizeURIPath` only mutates on
  `runtime.GOOS == "windows"`. The Windows branch checks for
  `[\\/][A-Za-z]:[\\/]` at the start of the path and strips the leading
  separator. Correct handling of the `Path()` quirk; the unit tests at
  `edit_test.go:24` cover both the `/C:/...` and `\C:\...` cases under a
  `runtime.GOOS != "windows"` skip guard.
- `internal/lsp/handlers.go:48` — `HandleApplyEdit` signature changes to
  accept `workspaceRoot string`. The single caller at
  `internal/lsp/client.go:198` passes `c.cwd`. As long as `Client.cwd` is
  set at construction time and never re-pointed to something outside the
  workspace, this is safe; if `cwd` is ever empty, `validateWorkspacePath`
  would treat the empty string's `filepath.Abs` as the process CWD, which
  is probably not what callers want — a one-line non-empty assertion in
  `validateWorkspacePath` (or in `client.go` at handler-registration time)
  would be belt-and-suspenders.
- `internal/lsp/util/edit.go:240,259,266` — rename and delete paths now
  validate both `OldURI` and `NewURI` (rename) and the target URI
  (delete). The recursive-delete test at `edit_test.go:493` confirms a
  recursive delete against an out-of-workspace path is rejected before
  any filesystem call. This closes the highest-impact vector — a recursive
  delete pointed at `/` would have been catastrophic under the old code.
- `internal/lsp/util/edit_test.go:411` — the `BoundaryEnforcement` test
  block has positive coverage (allows in-workspace edits) and three
  negative cases (text edit, recursive delete, rename — though the rename
  case is below the diff window I read). The error string match
  `"outside workspace root"` is a brittle but acceptable assertion shape.

## Verdict

`merge-after-nits`

## Rationale

This is a real, important defense-in-depth fix and the implementation is
mostly right. Two small things before merge: (1) confirm
`fsext.HasPrefix` enforces a path-component boundary (not naive string
prefix) — otherwise a workspace at `/home/user/proj` would accept edits
to `/home/user/proj-evil/file.txt`; (2) reject empty
`workspaceRoot` explicitly inside `validateWorkspacePath` so a
mis-wired caller can't accidentally degrade the check to "anywhere under
process CWD." Both are one-line additions and would close the remaining
edge cases without changing the design.
