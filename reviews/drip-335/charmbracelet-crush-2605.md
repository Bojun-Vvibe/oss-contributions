# Review: charmbracelet/crush #2605

- **Title:** feat(config): add additional_dirs option for tool access
- **Head SHA:** `e8e6bd2683ffc1257db9fc21673ce16db168e5ca`
- **Size:** +350 / -23
- **Files touched (sample):**
  - `internal/agent/agentic_fetch_tool.go`
  - `internal/agent/common_test.go`
  - `internal/agent/coordinator.go`
  - `internal/agent/tools/bash.go`
  - `internal/agent/tools/bash_test.go`
  - `internal/agent/tools/dircheck.go` (new)
  - `internal/agent/tools/dircheck_test.go` (new)
  - `internal/agent/tools/download.go`

## Critique

Adds an `additional_dirs` config option plus a `restrict_to_project` flag, plumbed through a new `tools.DirRestrictions` struct that is now a required argument to `NewBashTool`, `NewDownloadTool`, `NewEditTool`, `NewMultiEditTool`, `NewLsTool`, `NewViewTool`, `NewWriteTool`. New `dircheck.go` houses the policy.

Specific lines:

- `coordinator.go:464-481` — environment variable expansion in additional dirs uses `home.Long(dir)` followed by a `strings.HasPrefix(expanded, "$")` check and `Resolver().ResolveValue`. This means `~/foo` is expanded but `$HOME/foo` only works if `home.Long` doesn't already handle it. The two-step is fragile; consider a single canonicalization helper. Also, what about `${HOME}/foo` (curly form)? Not handled.
- `bash.go:200-205` — `absExecDir, _ := filepath.Abs(execWorkingDir)` silently drops the error. If `Abs` fails (very rare on Unix, more common on Windows with weird paths), the restriction check runs against an empty string and `DenyIfRestricted` may incorrectly allow. Surface the error.
- `coordinator.go:484-503` — every tool constructor now takes `dirRestrictions` as a positional arg. This is a wide signature change that touches `agentic_fetch_tool.go:172` (passes empty `tools.DirRestrictions{}`) and `common_test.go` (also empty). Empty `DirRestrictions{}` should be the documented "no restrictions" zero value — verify `DenyIfRestricted` returns `nil` for the zero value, otherwise existing tests will start failing in CI.
- `agentic_fetch_tool.go:172` passes `tools.DirRestrictions{}` to the View tool inside the agentic fetch sandbox. That's correct (the sandbox has its own tmpDir), but it means the sandbox tools intentionally bypass the user's `additional_dirs` policy. Document this explicitly — security reviewers will flag it.
- New `dircheck.go` (only first 6 lines visible in sample) — full file should be reviewed for path-traversal handling: are `..` segments resolved, are symlinks followed via `EvalSymlinks`, do `WorkingDir`/`AdditionalDirs` get cleaned with `filepath.Clean`? If symlinks aren't resolved, a user could create `~/proj/escape -> /etc` and bypass restrictions.
- The signature break is API-breaking for any out-of-tree code that constructs tools directly. Worth a CHANGELOG entry.

## Verdict

`merge-after-nits`
