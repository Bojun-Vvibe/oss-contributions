# charmbracelet/crush PR #2749 — fix(lsp): respect SingleFileSupport so single-file servers start without root markers

- URL: https://github.com/charmbracelet/crush/pull/2749
- Head SHA: `9bc7d24e3c471df6ca30b4c8e8eaa1d64229e808`
- Author: georgeglarson
- Verdict: **merge-as-is**

## Summary

Two-part fix in `internal/lsp/manager.go`:

1. When the user overrides a *known* LSP server (e.g. defines `pyright` with a custom `command:`), the previous code zeroed out powernap's curated capability flags (`SingleFileSupport`, `EnableSnippets`) because `manager.AddServer(name, &ServerConfig{…})` constructed a fresh struct. The fix re-reads the existing default via `manager.GetServer(actualName)` and copies those two flags forward.
2. The `handles(server, filePath, workDir)` predicate, which previously required `hasRootMarkers(workDir, server.RootMarkers)`, is relaxed to `(server.SingleFileSupport || hasRootMarkers(...))`. This matches the LSP spec: servers that advertise single-file support don't need a project root.

## Line-level observations

- `internal/lsp/manager.go` lines ~52–67: the refactor from "build the struct inline inside `AddServer`" to "build, then patch from defaults, then `AddServer`" is minimal and well-commented. The fallback uses `if defaultCfg, ok := manager.GetServer(actualName); ok { ... }` so unknown user-defined servers (no default) gracefully keep zero values — correct.
- Line 359: `hasRootMarkers(workDir, server.RootMarkers)` becomes `(server.SingleFileSupport || hasRootMarkers(workDir, server.RootMarkers))`. Short-circuits `SingleFileSupport=true` before doing filesystem walks — small perf win on top of the correctness fix.
- New test file `internal/lsp/manager_singlefile_test.go`:
  - `TestHandlesRespectsSingleFileSupport` exercises both branches (single-file true → handled, false → not handled) on the same temp file, asserting the predicate correctly. Good.
  - `TestNewManagerPreservesDefaultSingleFileSupport` constructs a `Config` with only `Command: "pyright-langserver"` (no `SingleFileSupport`) and verifies the registered server still has `SingleFileSupport=true`. This pins the regression.

## Why merge-as-is

- Scope is tight, both changes are necessary together, both have direct test coverage.
- Comment on lines ~60–62 ("Preserve capability flags from defaults … those are populated by powernap's curated allowlists") explains *why* clearly enough that a future maintainer won't undo it.
- Test uses `t.Parallel()` and `require` — consistent with repo style.

## Suggestions (optional, post-merge)

1. The set of "fields to preserve from defaults" is currently two (`SingleFileSupport`, `EnableSnippets`). If powernap ever adds more capability-flag fields, this code will silently regress for them. Worth a TODO or a small allowlist test that fails when new boolean fields appear in `powernapconfig.ServerConfig`.
2. Consider also preserving `RootMarkers` from defaults when the user override doesn't set them — same class of "user override clobbers curated metadata" bug.
