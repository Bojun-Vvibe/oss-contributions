# charmbracelet/crush PR #2790 — fix: hooks silently fail on Windows due to missing sh

- Link: https://github.com/charmbracelet/crush/pull/2790
- SHA: `358d5271f5986815d31855c2798cc00cd5adb582`
- Author: ilgax
- Stats: +52 / −1, 2 files

## Summary

Closes #2789. Replaces the unconditional `exec.CommandContext(ctx, "sh", "-c", hook.Command)` in `internal/hooks/runner.go` with a `pickShell(command)` helper that selects `sh -c` on non-Windows, and on Windows routes `.bat`/`.cmd` through `cmd /c`, `.sh` through `sh -c`, and falls back to `sh` if available else `cmd /c`. Adds a `skipIfNoSh(t)` guard to every Unix-shell test in `hooks_test.go` so the existing suite still passes on Windows runners that lack `sh`.

## Specific references

- `internal/hooks/runner.go` L150–L151: `cmd := exec.CommandContext(ctx, "sh", "-c", hook.Command)` becomes `shell, shellArg := pickShell(hook.Command); cmd := exec.CommandContext(ctx, shell, shellArg, hook.Command)`. Single hot-path call; minimal overhead since `pickShell` short-circuits to `("sh", "-c")` on non-Windows.
- `internal/hooks/runner.go` L218–L246 (new `pickShell`): the first-token extraction handles three cases: leading-quoted path (`"C:/path/with spaces.bat" arg`), unquoted whitespace-split, and the bare command. The `strings.Index(command[1:], "\"")` returns the inner offset (correct), but the resulting `first` slice is `command[1 : end+1]` — that's the substring between the first and second `"`, which is the intended path. Worth a unit test for `pickShell("\"C:/Program Files/foo.cmd\" arg")`.
- L237–L246: switch on `strings.ToLower(filepath.Ext(first))`:
  - `.bat`, `.cmd` → `cmd /c` (correct; `sh` cannot exec batch files).
  - `.sh` → `sh -c` (correct; assumes Git Bash / WSL on PATH).
  - default → `sh` if `exec.LookPath("sh") == nil` else `cmd /c`. Note this calls `exec.LookPath("sh")` on every hook execution on Windows; cache to a `sync.Once` if hook throughput matters. For typical interactive use this is negligible, but a tight loop of hook invocations would feel it.
- L237 `filepath.Ext(first)`: if the user wrote a bare `echo hello`, `Ext` returns `""` and the default branch fires — correct.
- L244 fallback to `cmd /c` for an arbitrary command string is a behaviour change worth documenting in the README/CHANGELOG: a Windows user with no `sh` on PATH who has been writing `echo '{"decision":"allow"}'` style hooks will now get those routed to `cmd /c`, where single-quoted JSON does not parse the same way. The PR fixes the silent-fail symptom but introduces a quoting-semantics surprise. Recommend either: (a) emit a one-time warning when falling back to `cmd /c`, or (b) document explicitly in the hook docs that Windows users without Git Bash should use `cmd`-compatible quoting.
- `internal/hooks/hooks_test.go` L13–L18: new `skipIfNoSh(t)` plus per-test `skipIfNoSh(t)` calls in `TestRunnerExitCode0Allow`, `TestRunnerExitCode2Deny`, `TestRunnerExitCode49Halt`, `TestRunnerHaltViaJSON`, `TestRunnerExitCodeOtherNonBlocking`, `TestRunnerTimeout`, `TestRunnerDeduplication`, `TestRunnerNoMatchingHooks`, `TestRunnerMatcherFiltering`, `TestRunnerParallelExecution`, `TestRunnerEnvVarsPropagated`, `TestRunnerUpdatedInput`. Mechanical and correct. No new positive Windows test was added — consider a Windows-only test that verifies `pickShell("foo.bat") == ("cmd", "/c")`.

## Verdict

verdict: merge-after-nits

## Reasoning

The core fix is right and the silent-fail symptom on Windows is real. Two non-blocking nits before merge: (1) add a `pickShell` unit test (cross-platform, just exercises the function directly) covering quoted paths, `.bat`/`.cmd`/`.sh`/bare cases — this is the new logic and it currently has no direct coverage; (2) document the `cmd /c` fallback behaviour in the hook docs/CHANGELOG since users with shell-style quoting will hit a different parser. The `exec.LookPath` on every hook on Windows is a minor concern only at high throughput.
