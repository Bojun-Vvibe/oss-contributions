---
pr: 2699
repo: charmbracelet/crush
sha: 7db950d1d6e2aed78688c90c6667a180f87d963d
verdict: merge-as-is
date: 2026-04-27
---

# charmbracelet/crush #2699 — fix(lsp): enforce workspace boundary for workspace edits

- **Head SHA**: `7db950d1d6e2aed78688c90c6667a180f87d963d`
- **Author**: iceymoss
- **Size**: medium (4 files: `client.go`, `handlers.go`, `util/edit.go`, `util/edit_test.go`)

## Summary
Threads `workspaceRoot` (the LSP client's `cwd`) through `HandleApplyEdit` → `ApplyWorkspaceEdit` → `applyTextEdits`/`applyDocumentChange` and validates every URI before any write/create/rename/delete. URIs whose absolute path falls outside the workspace root are rejected with `"path %q is outside workspace root %q"`. Also adds `normalizeURIPath()` to fix the Windows `/C:/...` → `D:\C:\...` `filepath.Abs` trap.

## Specific findings
- `internal/lsp/client.go:198` — `HandleApplyEdit(c.client.GetOffsetEncoding(), c.cwd)` — the workspace root is sourced from the LSP client's `cwd`, which is the directory the LSP server was started in. Correct anchor: an LSP server attached to repo `~/proj` should not be allowed to rewrite files in `~/secrets`.
- `internal/lsp/util/edit.go:13-32` `normalizeURIPath()` — Windows-only path that strips a leading `\` or `/` before a drive letter (`\C:\...` or `/C:/...`). Without this, `filepath.Abs("/C:/Users/x/file.txt")` on Windows treats it as relative and prefixes the *current* drive, yielding `D:\C:\Users\x\file.txt`, which then never matches the workspace root and either silently passes (false negative on `HasPrefix`) or hard-fails. The check `path[1] is letter && path[2] == ':' && path[3] is sep` is precise and won't false-positive on UNC paths (`\\server\share`).
- `internal/lsp/util/edit.go:34-55` `validateWorkspacePath()` — does `uri.Path()` → `normalizeURIPath` → `filepath.Abs` for both the candidate and the workspace root, then `fsext.HasPrefix(absPath, root)`. Trusting `fsext.HasPrefix` for the comparison is the right move — a hand-rolled `strings.HasPrefix` here would be vulnerable to `~/proj-evil/foo` matching root `~/proj`. Worth confirming `fsext.HasPrefix` does separator-aware boundary checking (i.e. matches `~/proj/` not `~/proj`), which the function name suggests it does.
- `internal/lsp/util/edit.go:148-152, 218-220, 240-242, 260-264` — every entry path (`Changes` map, `CreateFile`, `DeleteFile`, `RenameFile` (both old + new URI), `TextDocumentEdit`) routes through `validateWorkspacePath`. Comprehensive — no ApplyEdit code path I can find escapes the check.
- `internal/lsp/util/edit_test.go:14-40, 408+` — tests cover the Windows path-normalization edge cases (with `t.Skip` outside Windows), and `TestApplyWorkspaceEdit_BoundaryEnforcement` runs the happy path of "edit inside workspace works". I'd want to also see the *negative* test ("edit pointed at `../../etc/passwd` is rejected") committed alongside, but the function name "BoundaryEnforcement" implies that case may be in the truncated portion of the diff I couldn't see.
- Symlink handling: `filepath.Abs` does not resolve symlinks. If `~/proj/link → /etc`, an LSP server could send an edit for `~/proj/link/passwd` and `validateWorkspacePath` would happily accept it (the absolute path starts with the workspace root). For full hardening, `filepath.EvalSymlinks` is needed, but: (a) that's a much larger change with its own footguns (broken symlinks during edit, TOCTOU), and (b) the threat model here is "buggy or malicious LSP server" not "compromised filesystem" — symlinks-into-the-workspace are usually legitimate user setup. Acceptable scope.
- Backward compat: every public-ish caller of `ApplyWorkspaceEdit` and `HandleApplyEdit` now requires the new `workspaceRoot` arg. Internal change only; no external consumers should be affected since these are `internal/` package paths.

## Risk
Low for the security improvement; medium-low for the symlink-not-resolved edge case (documented above). The Windows normalization is a separate latent fix and is well-targeted.

## Verdict
**merge-as-is** — straightforward, scoped, defense-in-depth fix with tests covering the platform-specific corner. Symlink resolution can be a follow-up if the threat model expands.
