---
pr: 2699
repo: charmbracelet/crush
sha: 7db950d1d6e2aed78688c90c6667a180f87d963d
verdict: merge-after-nits
date: 2026-04-26
---

# charmbracelet/crush #2699 — fix(lsp): enforce workspace boundary for workspace edits

- **URL**: https://github.com/charmbracelet/crush/pull/2699
- **Author**: iceymoss
- **Head SHA**: 7db950d1d6e2aed78688c90c6667a180f87d963d
- **Size**: +202/-16 across 4 files (`internal/lsp/{client.go,handlers.go,util/edit.go,util/edit_test.go}`)

## Scope

Hardens `workspace/applyEdit` LSP request handling so that filesystem mutations (text edits, file creates, file deletes including `recursive: true`, file renames) are constrained to the configured workspace root.

Wiring changes:
- `client.go:198` — `HandleApplyEdit(c.client.GetOffsetEncoding(), c.cwd)` (passes workspace root through registration).
- `handlers.go:48-55` — `HandleApplyEdit(encoding, workspaceRoot string)` and forwards `workspaceRoot` into `util.ApplyWorkspaceEdit`.
- `edit.go:316,326,336` — `ApplyWorkspaceEdit` / `applyTextEdits` / `applyDocumentChange` all gain `workspaceRoot string` as a required parameter.

Core validator: `validateWorkspacePath` at `edit.go:24-46` — resolves URI to OS path, normalizes (with Windows-specific `\C:\…` / `/C:/…` handling at `edit.go:13-22`), absolutizes both the path and the root, and rejects with `path %q is outside workspace root %q` if `fsext.HasPrefix(absPath, root)` is false.

Boundary tests in `edit_test.go` cover: edits inside workspace allowed; text edits outside rejected; recursive delete outside rejected; rename to outside rejected.

## Specific findings

- **Threat model is real and underspecified in the LSP spec.** A malicious or misbehaving language server can return arbitrary `DocumentURI` values in `workspace/applyEdit` responses, and the LSP spec does not require the client to validate them. Crush previously took the server at its word, which is exploitable: a compromised npm-published `gopls`-shaped package, or a malicious `tsserver` plugin loaded by user config, could return `file:///etc/sudoers` in a rename and Crush would happily move it. This PR closes that.

- **`normalizeURIPath` Windows handling at `edit.go:13-22` is the most subtle part.** The comment correctly identifies that LSP `DocumentURI.Path()` on Windows returns `\C:\Users\...` or `/C:/Users/...` (leading separator before drive letter), and `filepath.Abs("\\C:\\Users\\foo")` would resolve to `D:\C:\Users\foo` (treating the leading `\` as relative-to-current-drive). The strip-leading-separator-when-followed-by-drive-letter logic is correct. One concern: the check at `:18-21` requires the byte at index `[3]` to be `\\` or `/` — i.e. it only strips when the path is at least `[sep][letter][:][sep]…`. A bare `\C:` (4 chars but no trailing sep) wouldn't be stripped. That's probably fine because `Abs("\C:")` would still produce `D:\C:` which then fails the `HasPrefix` check, so it falls into reject-by-default. Worth a one-line comment noting "if the URI is malformed enough to skip this strip, the boundary check will reject it — fail closed".

- **`fsext.HasPrefix(absPath, root)` is doing the right comparison** (assumed to be a path-segment-aware prefix check, not a string `strings.HasPrefix`). The latter would have a classic `/foo/barbaz` matches `/foo/bar` symlink-confusion bug. Reviewer should confirm `fsext.HasPrefix` actually does segment-aware comparison; if it's `strings.HasPrefix`, the check is exploitable.

- **Symlink resolution is missing.** `filepath.Abs` does not follow symlinks. If the workspace contains a symlink `inside_workspace -> /etc/sudoers`, an edit to `inside_workspace` would absolutize to a path under the workspace root (passes the check) but actually mutate `/etc/sudoers`. The fix is `filepath.EvalSymlinks` after `Abs` for both paths, or — more conservatively — reject symlinks under the workspace root during validation. Worth at least a TODO + follow-up issue. Not a blocker for this PR (it's still strictly more secure than before), but it's the next attack vector someone will report.

- **`HandleApplyEdit` returns `Applied: false, FailureReason: err.Error()`** at `handlers.go:55-56` (per the visible code). That's the right LSP response shape for a refused edit — the server gets a structured "no" rather than a connection drop, and language servers handle it gracefully. Good.

- **No change to `ApplyWorkspaceEdit`'s loop semantics.** If one of N edits fails the boundary check, the function returns immediately — earlier edits in the loop have already been applied to disk. That's a partial-application footgun. Not introduced by this PR (the previous code had the same all-or-nothing failure mode against arbitrary errors), but worth filing as a separate issue: ideally, validate *all* paths in a `WorkspaceEdit` before applying *any*. Two-phase commit shape.

- **Test coverage is appropriately focused** on boundary scenarios: edits-allowed-inside, text-edit-rejected-outside, recursive-delete-rejected-outside, rename-target-rejected-outside. Missing: rename-source-inside-target-outside (covered? need to see full test file), and the symlink case (not covered, see above).

- **`TestNormalizeURIPath` at `edit_test.go:14-34`** is gated behind `runtime.GOOS != "windows"` for the slash-separated path test (visible in the partial diff). The Windows-specific branches need their own test. Likely they're below the cutoff in the truncated diff — reviewer should spot-check.

- **Author drive-by quality is good.** Issue not linked but PR body has clear "Why / What / Test plan / Impact" sections, identifies the pre-condition ("filesystem operations from LSP workspace edits could be applied without strict workspace boundary enforcement"), and the test plan explicitly mentions `go test ./internal/lsp/...` + full suite.

## Risk

Low for the existing happy path: in-workspace edits, creates, deletes, and renames continue to work unchanged. The risk surface is operator-friction: language servers that legitimately need to edit files outside the workspace (e.g. monorepo language servers spanning multiple workspaces) will now be rejected. That's a feature, not a bug, but it should be documented somewhere (release notes? `internal/lsp/util/edit.go` package doc?).

## Nits

1. Confirm `fsext.HasPrefix` is segment-aware, not `strings.HasPrefix`. If the latter, this PR has a hidden traversal hole.
2. Add a TODO + follow-up issue for symlink resolution (`filepath.EvalSymlinks` post-`Abs`).
3. File a separate issue for two-phase application semantics so partial application can't happen on a multi-edit `WorkspaceEdit`.
4. One-line comment at `normalizeURIPath:18-21` noting the "fail closed if malformed" property.
5. Add a Windows-specific subtest for `normalizeURIPath` with `\C:\…` and `/C:/…` inputs (verify it's there in the un-truncated test file).
6. Surface a release-note entry that out-of-workspace edits are now rejected by design.

## Verdict

**merge-after-nits** — closes a real attack vector with the right architectural shape (validator passed top-down through the LSP handler chain, fail-closed default, structured LSP rejection rather than connection drop). The four-test boundary suite is appropriately focused. Symlink and two-phase-commit gaps are follow-up work, not blockers, but the `fsext.HasPrefix` assumption needs to be confirmed before merge.

## What I learned

LSP's `workspace/applyEdit` is one of the highest-trust surfaces in the protocol: a misbehaving server can rewrite the user's filesystem with no authentication, no sandboxing, and no spec-mandated client-side check. Every LSP client should have a boundary validator like this one — and the right place is at the top of the handler, with the workspace root threaded down as a required parameter (not a global, not a thread-local) so a future "convenience overload" can't accidentally call the validator-less path. The Windows `\C:\…` URI quirk is the kind of thing you only learn by deploying to Windows users and getting bug reports — worth saving as a generalized snippet for any LSP client implementation.
