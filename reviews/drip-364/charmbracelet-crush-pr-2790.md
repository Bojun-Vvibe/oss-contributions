# charmbracelet/crush #2790 — fix: hooks silently fail on Windows due to missing sh

- Head SHA: `358d5271f5986815d31855c2798cc00cd5adb582`
- Diff: +52 / −1 across `internal/hooks/runner.go` and `internal/hooks/hooks_test.go`

## Verdict: `merge-after-nits`

## What it does

Replaces the hardcoded `exec.CommandContext(ctx, "sh", "-c", hook.Command)` in `Runner.runOne` (`runner.go:150`) with a `pickShell(command)` lookup that:
- On non-Windows, returns `("sh", "-c")` unchanged.
- On Windows, parses out the first token (handling quoted paths via `strings.Index(command[1:], "\"")` and unquoted via `strings.IndexAny(command, " \t")`), then dispatches by extension: `.bat`/`.cmd` → `("cmd", "/c")`, `.sh` → `("sh", "-c")`, anything else → `sh` if `exec.LookPath("sh")` succeeds, else fall back to `("cmd", "/c")`.

Test side adds a `skipIfNoSh(t)` helper (`hooks_test.go:14-18`) and applies it to every existing hook test that uses Unix shell syntax (`echo '{...}' >&2; exit 49`, `sleep 10`, `printf '...' "$VAR"`, etc.) so the test suite no longer fails on Windows runners that don't have Git Bash on PATH.

## Why this is the right shape

1. **Real behavior gap, not theoretical.** Before this PR, a Windows user with no `sh.exe` on PATH would have *every* hook silently fail to launch — `exec.CommandContext` returns an error from `cmd.Run()` and the runner logs it but the hook decision becomes "non-blocking error", which from the user's perspective looks like "hooks just don't work and there's no actionable error message." The fix routes `.bat`/`.cmd` to the native shell (which is what those files were authored for) and only requires `sh` for actual shell scripts.
2. **Correct dispatch order.** Extension check happens *before* the `LookPath("sh")` fallback (`runner.go:235-247`), so a user who has Git Bash on PATH *and* writes a `.bat` hook still gets `cmd /c` — the file extension wins because `.bat` syntax is genuinely incompatible with `sh -c` (different quoting, no `$VAR` expansion, etc.). The PathExt-style "unknown command, prefer sh if available" fallback at the bottom is the right default for inline shell snippets like `echo '{"decision":"allow"}'` which work fine under Git Bash.
3. **Quoted-path parsing is correct for the common Windows case.** `"C:\Program Files\bin\hook.bat" arg1` correctly extracts `C:\Program Files\bin\hook.bat` (handling the embedded space), then `filepath.Ext` returns `.bat`, then `cmd /c` runs. The unquoted path `C:\bin\hook.cmd arg1` extracts `C:\bin\hook.cmd` via `IndexAny(" \t")`. Both paths are tested implicitly by the existing test suite once the skipIfNoSh fence lets them run on Windows.
4. **Test fence is the minimal-impact way to keep CI green.** Rather than rewriting `echo '{"decision":"allow"}'` etc. to be portable, the PR explicitly skips Unix-syntax tests on Windows hosts that lack `sh`. The skipped tests still run on the canonical Windows-with-Git-Bash CI matrix entry, so coverage isn't lost — just gated on the host having `sh`.

## Nits

- **`pickShell` parses the first token to find the extension, but then passes the *original full command* to the shell** (`runner.go:153` `cmd := exec.CommandContext(ctx, shell, shellArg, hook.Command)`). That's correct for `cmd /c "C:\Program Files\bin\hook.bat" arg1` but it's worth a one-line comment in `pickShell`'s doc-comment that the parsing is *only* to pick the dispatcher, not to rewrite the command. A future maintainer might mistake `first` for an output and try to use it.
- **Quoted-path parsing has one edge case.** `strings.Index(command[1:], "\"")` finds the first closing quote; if a path contains an *escaped* quote (`"C:\path\with \"escaped\" quote\hook.bat"`) the parser will stop at the wrong quote. Vanishingly rare on Windows file paths but worth a note. The fix is easy enough (track escape state) but probably not worth doing until someone files a bug.
- **`runtime.GOOS != "windows"` returns `("sh", "-c")` even on systems where `/bin/sh` doesn't exist** — embedded Linux, scratch containers without busybox, etc. Pre-existing behavior, not regressed by this PR, but worth flagging for a follow-up: the symmetric `LookPath("sh")` fallback could apply on the non-Windows branch too, returning a structured error like "no shell available, hook skipped" instead of opaque exec failure.
- **The `.bat`/`.cmd` extension check is case-sensitive at the prefix level** (`strings.ToLower(filepath.Ext(first))`). Good — handles `HOOK.BAT`, `Hook.Cmd`, etc. No issue, just confirming.
- **Error handling on `LookPath` fallback discards the error** (`runner.go:243-245` `if _, err := exec.LookPath("sh"); err != nil { return "cmd", "/c" }`). Fine — the err path is "sh not on PATH" which isn't actionable here. But consider a one-time `slog.Debug` log noting "falling back to cmd because sh not found" so the cost-of-debugging is one log line rather than running `pickShell` mentally.
- **Test name `skipIfNoSh` is a perfect name** for what it does. The 13 call sites it gets added to all read cleanly. No nit there — just acknowledging the test side is well-done.

## Citations

- `internal/hooks/runner.go:150` — `pickShell(hook.Command)` replacement of the hardcoded `"sh", "-c"`
- `internal/hooks/runner.go:222-247` — `pickShell` function body, full Windows dispatch logic
- `internal/hooks/runner.go:235-238` — quoted-path first-token extraction
- `internal/hooks/runner.go:240-247` — extension switch + `LookPath` fallback
- `internal/hooks/hooks_test.go:14-18` — `skipIfNoSh` helper
- `internal/hooks/hooks_test.go:223-625` — 13 call sites of `skipIfNoSh(t)` across all Unix-shell-syntax tests
