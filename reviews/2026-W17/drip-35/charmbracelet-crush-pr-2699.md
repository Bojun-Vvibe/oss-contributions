# charmbracelet/crush #2699 — fix(lsp): enforce workspace boundary for workspace edits

- **Repo**: charmbracelet/crush
- **PR**: [#2699](https://github.com/charmbracelet/crush/pull/2699)
- **Head SHA**: `7db950d1d6e2aed78688c90c6667a180f87d963d`
- **Author**: iceymoss
- **State**: OPEN (+202 / -16)
- **Verdict**: `merge-after-nits`

## Context

Before this PR, `util.ApplyWorkspaceEdit` accepted whatever URIs the LSP
server handed back and applied them directly via `os.Remove`,
`os.RemoveAll`, `os.Rename`, and `os.WriteFile`. There is no LSP-protocol
guarantee that those URIs sit inside the project root, so a hostile or
buggy server could mutate `~/.ssh`, `/etc`, or any path the agent process
can write. This is exactly the kind of capability boundary an agentic LSP
client must enforce locally — the spec deliberately leaves it to the
client.

## Design

The patch threads `workspaceRoot` from `Client.cwd` all the way into
`ApplyWorkspaceEdit`, then funnels every URI through a single
`validateWorkspacePath` helper at `internal/lsp/util/edit.go:25-46`:

```go
absPath, err := filepath.Abs(path)
...
root, err := filepath.Abs(workspaceRoot)
...
if !fsext.HasPrefix(absPath, root) {
    return "", fmt.Errorf("path %q is outside workspace root %q", absPath, root)
}
```

That `fsext.HasPrefix` is the right primitive (it is presumably the
project's existing prefix check that handles trailing-slash and case
folding correctly — much better than a naive `strings.HasPrefix(absPath,
root)` which would match `/work` against `/workshop`). All four
mutation paths in `applyDocumentChange` (create / delete / rename old /
rename new) and the `applyTextEdits` entry route now validate, so there
is no asymmetric leak.

The Windows-specific `normalizeURIPath` at lines 13-32 is the part I'd
look at hardest. The LSP `DocumentURI.Path()` quirk on Windows is real
(`\C:\…` vs `C:\…`), and `filepath.Abs` on `\C:\…` does turn into
`<cwd-drive>:\C:\…`. The strip is bounded (`len(path) >= 4`,
ASCII letter, colon, separator) so it can't accidentally chew off a leading
separator from a non-drive path. Good.

## Risks / Nits

1. **Symlink escape is still possible.** `filepath.Abs` does not resolve
   symlinks. If a workspace contains `link -> /etc`, an LSP edit to
   `link/passwd` will pass the boundary check (`absPath` stays inside
   the workspace) and then `os.WriteFile` follows the symlink out. The
   defense-in-depth fix is `filepath.EvalSymlinks` on `absPath` before
   the prefix check, mirroring what skills #2694 just did for dedup.
   Strongly recommend this as a follow-up — without it, "boundary
   enforcement" is technically true but practically bypassable on any
   repo that has a symlinked vendor dir or a malicious `git clone`.

2. **Symlink on the workspace root itself.** If `c.cwd` is itself a
   symlink (common on macOS with `/var` -> `/private/var`), the
   `filepath.Abs(workspaceRoot)` on one side and the resolved path on
   the other can mismatch. Consider `EvalSymlinks` on both sides
   together so they normalize to the same physical root, then compare.

3. **Test coverage is solid for the four happy/sad paths** added at
   `edit_test.go:408-515`. Missing: a TextDocumentEdit (the most common
   path through `applyDocumentChange`'s final branch at line 291) with
   an outside-workspace URI. The `Changes`-map path is covered, but the
   `DocumentChanges → TextDocumentEdit` path is not, and the dispatch
   to `applyTextEdits` is what catches it. Two-line addition.

4. **Error message leaks the workspace root** (`"path %q is outside
   workspace root %q"`). For an LSP client this is fine — operator
   sees their own root — but if these errors ever propagate into
   user-facing surfaces, consider stripping the second `%q`.

5. Nit: `validateWorkspacePath` returning `(string, error)` is good, but
   the return value is unused at the create/delete/rename call sites
   that immediately re-extract `path` from the URI separately. Actually
   they do use it — re-reading the diff at lines 240, 249, 264, 270 —
   the returned `path` flows into the subsequent `os.MkdirAll` /
   `os.RemoveAll` / `os.Rename` calls. Disregard.

## Verdict rationale

`merge-after-nits` rather than `merge-as-is` because (1) the symlink
gap is a real bypass of the stated security goal and (2) the
`TextDocumentEdit` test path is missing. Neither is a blocker for
merging — the PR is strictly better than `main` — but both should land
in a follow-up before this is advertised as a security fix.

## What I learned

This is a textbook example of "the LSP spec gives you a URI; treat it
as untrusted input." The fix is right, and threading the workspace root
through the handler signature instead of pulling it from a global is the
clean choice. The symlink hole is the standard second-step you have to
remember whenever you write a path-prefix sandbox in Go — `filepath.Abs`
is lexical, `EvalSymlinks` is the one that touches the filesystem.
