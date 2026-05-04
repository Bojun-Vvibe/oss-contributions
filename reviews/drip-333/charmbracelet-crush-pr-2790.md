# charmbracelet/crush PR #2790 — fix: hooks silently fail on Windows due to missing sh

- **Link:** https://github.com/charmbracelet/crush/pull/2790
- **Head SHA:** `358d5271f5986815d31855c2798cc00cd5adb582`
- **Verdict:** `merge-after-nits`

## What it does

Replaces the unconditional `exec.CommandContext(ctx, "sh", "-c", hook.Command)`
at `internal/hooks/runner.go:147` with a runtime-OS-and-extension-aware
`pickShell(command)` helper at `internal/hooks/runner.go:151-178`. On
non-Windows it remains `sh -c`; on Windows it dispatches `.bat`/`.cmd` to
`cmd /c`, `.sh` to `sh -c`, and everything else to `sh` if available else
`cmd /c`. Also adds a `skipIfNoSh(t)` guard (`hooks_test.go:14-18`)
sprinkled across 11 tests so CI on Windows-without-sh doesn't false-fail.

## Design analysis

- **Right diagnosis.** Hooks on Windows that rely on Git-Bash-installed `sh`
  worked silently; users without Git Bash got hook commands that exited
  with `exec: "sh": executable file not found in %PATH%` and the runner
  treated that as a non-blocking error (the existing behavior at the
  surrounding code path), so the user saw no hint that hooks weren't
  running at all. This PR makes hooks actually run under the OS-native
  shell as fallback.
- **Extension-driven dispatch is the right primitive on Windows.** The
  switch on `filepath.Ext(first)` at `runner.go:166-176` correctly
  distinguishes "user wrote a `.bat` script" from "user wrote a Unix
  `sh` one-liner". The `.sh` → `sh -c` branch keeps the door open for
  WSL/Git-Bash users to write portable hook scripts that work both ways.
- **Quote handling for the first token** at `runner.go:159-164` correctly
  parses a leading `"path with spaces.bat"` so the extension lookup hits
  the right token. Important on Windows where program paths often live
  under `C:\Program Files\…`.
- **`skipIfNoSh` is necessary but blunt.** All 11 modified tests use
  shell-specific syntax (`echo X >&2`, `exit 49`, `printf '%s' "$VAR"`,
  `sleep 10`). The skip is the right call rather than rewriting the test
  fixtures to PowerShell — those tests are testing the *shell exec* code
  path, not the hook policy machinery. Once `pickShell` lands, a
  follow-up PR could add a Windows-only test that uses `cmd /c echo` to
  cover the new branch, but that's a strict expansion of coverage, not a
  bug fix.

## Risks / nits

1. The "no extension, sh available" branch at `runner.go:172-175` calls
   `exec.LookPath("sh")` *on every hook invocation*. For a chatty hook
   surface (PreToolUse fires on every tool call), that's a stat per call.
   Cache the result behind a `sync.Once` at package init — the `PATH`
   doesn't change mid-process for this purpose.
2. `pickShell` returns `(string, string)` — readability would improve with
   a small typed struct `Shell{Bin, Arg string}` or even an enum. As-is
   the call site at `runner.go:148-149` is fine but the return contract
   isn't documented.
3. The Windows `cmd /c` branch will now invoke commands via `cmd.exe`,
   which has very different quoting rules than `sh -c`. A hook configured
   as `echo "Hello World"` will work in both shells; a hook configured as
   `for f in *.txt; do echo $f; done` will silently break on the cmd
   path. Worth a doc warning + a config validator that flags
   shell-specific syntax in hook commands.
4. The `.cmd`/`.bat` detection only checks the *first token*. A hook
   command like `wsl run-script.sh` (where `wsl` is the windows
   subsystem launcher) would incorrectly route to `sh -c` (or worse,
   `cmd /c` if no sh) when the user intended `cmd /c wsl run-script.sh`.
   Edge case but worth a comment.
5. `skipIfNoSh` is added but `TestRunnerNoMatchingHooks` at line 339
   doesn't actually invoke any shell command — the skip is unnecessary
   there (the test config is `nil` hooks). Harmless, just noise.

## What I learned

The "shell out via sh -c" anti-portability bug is universal across CLI
tools that spread to Windows after a Unix-first design. The right shape
of fix is *always* the OS-extension dispatch shown here rather than
"detect and rewrite the command" — the latter is a quoting-bug minefield.
The extension-dispatch shape leaves the user in control of which shell
they want by choosing the file extension, which is the cleanest possible
contract.
