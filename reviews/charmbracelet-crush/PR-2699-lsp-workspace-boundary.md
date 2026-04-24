# PR-2699 — fix(lsp): enforce workspace boundary for workspace edits

[charmbracelet/crush#2699](https://github.com/charmbracelet/crush/pull/2699)

## Context

LSP servers can return `workspace/applyEdit` requests that touch
arbitrary file paths. Until this PR, the handlers in `internal/lsp/
util/edit.go` (`applyTextEdits`, `applyDocumentChange`,
`ApplyWorkspaceEdit`) would resolve the URI's path and apply the
operation directly — including `RenameFile`, `DeleteFile` (with
`Recursive: true`), and `CreateFile` outside the workspace root.
A misbehaving or malicious language server could `rm -rf` adjacent
directories or write into the user's `~/.ssh`.

The PR threads a `workspaceRoot string` parameter through the four
edit functions, plus `HandleApplyEdit(encoding, workspaceRoot)` in
`handlers.go`, and wires it from `client.cwd` in `client.go`'s
`registerHandlers`. The new `validateWorkspacePath(uri, workspaceRoot)`
helper:

1. Resolves the URI to a path.
2. Calls `normalizeURIPath` (Windows: strips the leading `/` from
   `/C:/Users/test/file.txt`-style URI paths).
3. `filepath.Abs` on both path and root.
4. `fsext.HasPrefix(absPath, root)` check; reject otherwise.

Tests in `edit_test.go` (+124) cover: in-workspace edit succeeds;
text edit outside workspace rejected; recursive delete outside
workspace rejected (and the directory still exists after); rename
where target is outside the workspace.

## Strengths

- **The fix is in the right layer.** Validation is at the
  filesystem-mutation boundary, not at the JSON-RPC handler — so
  any future code path that calls `ApplyWorkspaceEdit` directly
  (e.g. a code-action runner) inherits the check.
- **Windows URI path quirk is handled deliberately.** The comment
  on `normalizeURIPath` explains *why* (`/C:/path` decoded from
  `file:///C:/path` would otherwise resolve to `D:\C:\...` on a
  machine with `D:` as cwd) — that's exactly the kind of platform
  fact that gets lost in code review without the comment.
- **Recursive delete test asserts the directory still exists after
  rejection** (`_, statErr := os.Stat(outsideDir); require.NoError
  (t, statErr)`). That's the correct assertion: it's not enough to
  return an error, the side effect must not have occurred.
- **Rename validates both `OldURI` and `NewURI`.** The PR could
  have validated only the destination; checking both is right
  because a malicious server could move a workspace file *out* of
  the tree just as easily as bringing one in.
- **Uses the existing `fsext.HasPrefix`** rather than a string
  `strings.HasPrefix`, which would be vulnerable to
  `/home/user/proj` prefixing `/home/user/project-other`.

## Concerns / risks

- **Symlink escape not addressed.** `filepath.Abs` does not resolve
  symlinks. If `workspaceRoot/link` is a symlink pointing at
  `/etc`, then `validateWorkspacePath("workspaceRoot/link/passwd",
  ...)` returns success and the edit lands at `/etc/passwd`.
  `filepath.EvalSymlinks` should be applied (with care: the path
  may not exist for `CreateFile`, in which case eval the parent
  directory and rejoin the basename).
- **`CreateFile` outside workspace is rejected,** which is right —
  but the test suite (in the diff snippet I have) doesn't appear
  to cover the `CreateFile` branch explicitly. The README lists
  4 cases (allow inside, reject text outside, reject delete
  outside, reject rename outside) — adding "reject create outside"
  would close the matrix.
- **`workspaceRoot` is taken from `c.cwd` once at handler
  registration.** If the user changes workspace mid-session
  (multi-root LSP, `workspace/didChangeWorkspaceFolders`), the
  registered handler keeps the old root. Worth making
  `workspaceRoot` a closure over a pointer/getter so the live
  value is read on each request, or re-registering on root change.
- **Multi-root workspaces are not modeled.** LSP supports multiple
  workspace folders; the current API takes a single string. If
  crush ever adds multi-root, this signature needs to become
  `[]string` and the prefix check becomes "is any root a prefix".
  Document the single-root assumption.
- **Error message leaks the absolute path:**
  `path %q is outside workspace root %q`. Fine for local
  debugging, but if these errors are ever surfaced to a remote
  LSP server (via `FailureReason` in `ApplyWorkspaceEditResult`),
  the absolute filesystem path of the user's home is disclosed
  back to the language server. Consider stripping to a relative
  hint or eliding.
- **`normalizeURIPath` runs `filepath.Clean(filepath.FromSlash(...))`
  on `workspaceRoot`** before the `filepath.Abs` call. `Abs` will
  re-clean. Cosmetic, not a bug.

## Verdict

**Request changes — small.** The boundary check is correct in shape
and the Windows path normalization is a thoughtful addition. Before
merge: add `EvalSymlinks` (or document the symlink-escape gap), add
the `CreateFile`-outside-workspace test case, and decide whether to
strip absolute paths from the error message returned via
`ApplyWorkspaceEditResult.FailureReason`. The multi-root concern
can ship as a follow-up issue.
