# charmbracelet/crush PR #2790

- **Title**: fix: hooks silently fail on Windows due to missing sh
- **Author**: ilgax
- **Head SHA**: `358d5271f5986815d31855c2798cc00cd5adb582`
- **Diff**: +52 / -1

## Summary

On Windows, `exec.CommandContext(ctx, "sh", "-c", hook.Command)` fails when `sh.exe` isn't on PATH (common — Git Bash not always installed). This PR introduces `pickShell()` to dispatch by extension on Windows: `.bat/.cmd` → `cmd /c`, `.sh` → `sh -c`, fall back to `sh` if available else `cmd /c`. Adds `skipIfNoSh` to existing tests so they don't fail on a sh-less Windows runner.

## File-by-file

### `internal/hooks/runner.go` (L130-178)

- **L140-141**: replaces hardcoded `"sh", "-c"` with `pickShell(hook.Command)`. Correct extraction point — single call site change.
- **L150-177 `pickShell`**:
  - L155-157: non-Windows always returns `sh -c`. Preserves prior behavior on Linux/macOS exactly. Good.
  - L158-166 first-token extraction: handles two cases — quoted path (`"C:\path with spaces\hook.bat"`) and space/tab-separated. Correctly bounds the inner quote scan with `command[1:]` then `+1` adjust at L162. Note: if the command starts with `"` but has no closing `"`, `first` stays as the entire command including the leading `"`. `filepath.Ext` on that returns `""` → falls through to default branch → either sh or cmd. Defensible degraded behavior.
  - L167-176 dispatch: `.bat`/`.cmd` → `cmd /c`, `.sh` → `sh -c`, default → sh-if-available else `cmd /c`. The default-to-sh-when-available preserves existing user expectations on Windows + Git Bash setups.
  - **Concern L173**: `exec.LookPath("sh")` runs *per hook invocation*. For a chatty hook setup this is a stat-per-PATH-entry every call. Cache once at init — trivial fix. Even better: cache the entire `(shell, shellArg)` tuple per `command` string.

### `internal/hooks/hooks_test.go`

- L17-21 `skipIfNoSh` helper: clean. Used for tests whose `Command` strings are clearly POSIX-shell (`echo`, `>&2`, `exit N`, `sleep`, `printf '%s'`).
- L30, L37, L46, L54, L62, L70, L78, L86, L94, L102, L110, L118 — all `t.Parallel()` test functions get the skip. Coverage is correct: every test that constructs a POSIX-shell-only `Command` is gated.
- One miss to double-check: `TestRunnerNoMatchingHooks` (L84-89) skips on no-sh but its hook list is empty — it never spawns a shell. Including the skip is harmless but technically unnecessary. Trivial.

## Risks

- `cmd /c "echo {...}"` semantics differ from `sh -c` for the JSON payload examples in the test bodies (quoting, `>&2` redirection works in cmd, but single-quoted JSON like `'{"decision":"allow"}'` would not strip quotes the same way). The skip-when-no-sh approach correctly sidesteps this; production users on Windows will need to author either `.bat` hooks or install Git Bash. PR doesn't add docs for this — would be valuable.
- A user script named `hook.sh.txt` will be matched by the default branch (no `.sh` ext detected) — correct behavior.

## Nits

- Cache `exec.LookPath("sh")` result on first call (L173).
- Add a doc note in the hooks docs that on Windows, hook commands without a `.bat`/`.cmd`/`.sh` suffix require `sh` (Git Bash / WSL / msys) on PATH.

## Verdict

**merge-after-nits**
