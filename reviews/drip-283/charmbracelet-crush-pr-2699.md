# charmbracelet/crush PR #2699 — fix(lsp): enforce workspace boundary for workspace edits

- Head SHA: `7db950d1d6e2aed78688c90c6667a180f87d963d`
- Size: +202 / -16, 4 files

## Specific refs

- `internal/lsp/util/edit.go:14-34` — new `normalizeURIPath` handles the Windows-specific case where `protocol.DocumentURI.Path()` returns `"\C:\Users\..."` or `"/C:/Users/..."`. `filepath.Abs` would otherwise treat these as relative and prefix the current drive, producing `"D:\C:\Users\..."`. The check at lines 24-28 strips a leading separator iff followed by `<letter>:<sep>`. Sound logic; the runtime check (`runtime.GOOS != "windows"`) keeps the fast path on Unix.
- `internal/lsp/util/edit.go:36-58` — new `validateWorkspacePath` resolves URI → abs path, resolves workspace root → abs, then `fsext.HasPrefix(absPath, root)` gate. Returns explicit "outside workspace root" error.
- `internal/lsp/util/edit.go:60-66, 218-244, 257-265` — `applyTextEdits`, `applyDocumentChange` (CreateFile, DeleteFile, RenameFile **both** OldURI and NewURI) all gated through `validateWorkspacePath`. The dual-URI rename gate is critical — validating only OldURI would still allow renames to escape the workspace.
- `internal/lsp/handlers.go:48` — `HandleApplyEdit` signature now takes `workspaceRoot string`; `internal/lsp/client.go:198` passes `c.cwd` from the wiring layer.
- `internal/lsp/util/edit_test.go` (+139) — boundary tests for in-workspace edit (allow), out-of-workspace text edit (reject), out-of-workspace recursive delete (reject), out-of-workspace rename target (reject).

## Assessment

Real security hardening with the right structural placement — gate at the apply layer, not the protocol parse layer, so any future code path that constructs an `ApplyWorkspaceEdit` is automatically covered. The Windows path quirk is genuinely subtle and worth the targeted normalization. `fsext.HasPrefix` is the correct primitive here (assuming it's a path-segment-aware prefix, not a string prefix — worth confirming it doesn't allow `/work` to match `/work-evil` via byte prefix).

Concerns:
- The rename test should explicitly cover the case where OldURI is *inside* the workspace but NewURI is *outside* (and vice versa) — the diff suggests both paths are validated but the test summary only mentions "rename when target is outside workspace".
- Symlinks: `filepath.Abs` does not resolve symlinks. A symlink inside the workspace pointing outside would still pass the prefix check on its lexical abs path. Worth a `filepath.EvalSymlinks` step or an explicit non-goal note.
- The change is a behavior-breaking change for any LSP server that legitimately needs to edit files outside the workspace (rare, but `gopls` workspace-folders setups exist). PR says "Expected behavior for valid in-workspace edits is unchanged" — true, but worth flagging in release notes.

verdict: needs-discussion
