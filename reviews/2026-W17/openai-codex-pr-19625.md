# openai/codex #19625 — Reset TUI keyboard reporting on exit

- **Repo**: openai/codex
- **PR**: #19625
- **Author**: etraut-openai (Eric Traut)
- **Head SHA**: 45b3851384fb73e83e4089cb6a091574e10ec585
- **Link**: https://github.com/openai/codex/pull/19625
- **Fixes**: #19553
- **Size**: 4 files, +328 / −195. Net: 280 lines moved into a new
  `tui/src/tui/keyboard_modes.rs` module, plus a new exit-path
  helper that emits stronger ANSI terminal-state resets.

## What it changes

1. **Module split**: all keyboard-enhancement-related code
   (`keyboard_enhancement_disabled_for`, `parse_bool_env`,
   `running_in_wsl`, `running_in_vscode_terminal`,
   `term_program_is_vscode`, `windows_term_program`,
   `read_windows_term_program`, plus their tests) moves out of
   `tui/src/tui.rs` into the new `tui/src/tui/keyboard_modes.rs`.
   `tui.rs` shrinks by 193 lines, the new module is 280 lines
   (the difference is two new ANSI command structs).
2. **New exit-only restore path**: `tui::restore_after_exit()`
   added (`tui.rs:200-208`) alongside the existing
   `tui::restore()`. Both call into a refactored `restore_common`
   (`tui.rs:160-194`) that now takes two enum params:
   `RawModeRestore` (Disable | Keep) and `KeyboardRestore`
   (PopStack | ResetAfterExit).
3. **The actual fix**: `reset_keyboard_reporting_after_exit()`
   in `keyboard_modes.rs:488-496` issues *three* sequenced
   commands instead of the old single `PopKeyboardEnhancementFlags`:
   - `PopKeyboardEnhancementFlags` (canonical kitty-protocol pop)
   - `ResetKeyboardEnhancementFlags` (`\x1b[<u`, clears *all*
     pushed levels — the hammer)
   - `DisableModifyOtherKeys` (`\x1b[>4;0m`, resets the older
     xterm `modifyOtherKeys` reporting that iTerm2 uses)
4. **Call-site swaps**: every place the TUI was tearing down on
   exit (panic hook at `tui.rs:293`, the top-level `restore()`
   in `lib.rs:1586` and `lib.rs:1605`) now calls
   `restore_after_exit()` instead. The non-exit `restore()` and
   `restore_keep_raw()` (used by, e.g., job-control suspend)
   keep the pop-only behaviour because the TUI re-enters with
   `set_modes()` shortly after.
5. **Two new ANSI command types** (`keyboard_modes.rs:498-538`):
   `ResetKeyboardEnhancementFlags` writing `\x1b[<u`, and
   `DisableModifyOtherKeys` writing `\x1b[>4;0m`. Both stub out
   `execute_winapi` with `ErrorKind::Unsupported` and set
   `is_ansi_code_supported() => false` on Windows so the legacy
   Windows console path skips them.
6. **`restore_common` error handling improved** (`:172-194`) —
   first error from `DisableBracketedPaste` is preserved; raw-
   mode-disable error is only kept if there wasn't an earlier
   one. Previously the function silently swallowed the bracketed-
   paste error if raw-mode disable also failed.

## Strengths

- **Diagnoses a real, well-understood problem precisely.** PR
  body cites the iTerm2 + Ctrl+C exit-without-stack-pop scenario
  as the reproducer for #19553. The "missing stack pop leaves
  parent shell receiving raw CSI-u fragments" is a class of bug
  that's hard to reproduce reliably; the fix is to fire the
  reset commands defensively at exit, on the correct theory that
  it's harmless if the stack already popped and curative if it
  didn't.
- **The two-axis enum refactor of `restore_common` is the right
  abstraction**. Before this PR, the `should_disable_raw_mode:
  bool` and the (always-the-same) keyboard-pop path were
  intertwined. Splitting into `RawModeRestore` × `KeyboardRestore`
  means future callers can mix and match without new boolean
  parameters or a fourth helper function.
- **ANSI command structs are real `Command` impls** with
  `execute_winapi` errors plus `is_ansi_code_supported() =>
  false` on Windows. That's the pattern crossterm uses; it
  means `execute!` will skip the writes on legacy Windows
  consoles instead of panicking. Tests at `:614-621` pin the
  exact ANSI strings (`\x1b[<u` and `\x1b[>4;0m`) so a
  refactor can't silently change the protocol.
- **Tests preserved across the module move** (`:568-621`),
  including the WSL/vscode auto-disable matrix and the
  env-flag override. Plus two new tests for the ANSI emission.
- **`restore` vs `restore_after_exit` distinction is
  documented** at `tui.rs:200-208`: "Uses a stronger keyboard
  reset than `restore` so the parent shell recovers even if a
  terminal missed the stack pop that normally pairs with
  `set_modes`." That's exactly the right comment to leave —
  future maintainers will see why there are two helpers.
- **Panic hook updated to use `restore_after_exit`**
  (`tui.rs:293`) — panics are exits, and the parent shell
  shouldn't be left in enhanced-keys mode just because the
  process crashed.

## Concerns / asks

- **The three resets fire in sequence with `let _ =
  execute!(...)` swallowing errors** (`keyboard_modes.rs:489-495`).
  If the first command (`PopKeyboardEnhancementFlags`) fails
  (e.g. the terminal doesn't support kitty protocol at all),
  the second and third still fire — which is the intended
  behaviour. But if the *write* itself fails (e.g. stdout is
  closed because the parent shell already exited), all three
  commands fail silently. Consider logging at debug level on
  the first failure so the rare "exit cleanup didn't actually
  happen" case is observable. `verbose_logger`/`tracing` debug
  is enough.
- **`DisableModifyOtherKeys` writes `\x1b[>4;0m`** — this is
  the xterm "reset to mode 0" sequence, which is correct for
  modifyOtherKeys, but on terminals that *don't* support
  modifyOtherKeys at all (e.g. very old VT-only emulators), the
  bare ANSI bytes will end up echoed into the parent shell as
  garbled input. Probably a non-concern in 2026, but worth a
  note in the docstring at `keyboard_modes.rs:498-516` saying
  "this assumes a vt100+ terminal that recognises the
  `\x1b[>...m` private-mode form".
- **No integration test** for the exit-path behaviour as a
  whole. The unit tests pin individual ANSI strings, but
  there's no test that asserts "after `restore_after_exit`,
  `restore_common` returns `Ok(())` even when push never
  happened". A simple one-line test using the existing
  `restore_after_exit()` directly under a `cargo test` runner
  would close this.
- **The module split is structurally a refactor + a feature in
  one PR**. The ~280-line move is mechanical, the ~50-line
  feature (the new resets and the two-axis enum) is the actual
  fix. Splitting them would make the diff easier to review,
  though for a TUI maintainer team that owns this code, the
  combined diff is probably fine.
- **`restore_common` error precedence: first error wins**
  (`:175-188`). The chosen precedence is bracketed-paste >
  raw-mode-disable. The keyboard-restore path errors are
  swallowed via `let _ =` in `keyboard_modes.rs`. If a debugger
  ever wants to know "what failed during exit", they'll see
  the bracketed-paste error but not the keyboard one. Worth a
  comment documenting why bracketed-paste is the priority
  (presumably: it's the visible-to-user one).
- **`#[cfg(target_os = "linux")] fn read_windows_term_program`
  via `cmd.exe`** moved into the new module unchanged. It
  spawns `cmd.exe /d /s /c set TERM_PROGRAM` on every WSL
  invocation. The `OnceLock` caches the result so it's
  one-shot per process — fine — but the cost on first call is
  a process spawn. Probably fine for a startup-time check, but
  a comment on the `OnceLock` saying "first-call cost is one
  cmd.exe spawn" would help.

## Verdict

**merge-after-nits** — the diagnosis is correct, the fix is
defensive in the right way, and the refactor that enables it
(`restore_common`'s two-axis enum) is a structural
improvement that pays for itself. The asks are: a debug-log
on the keyboard-restore swallow path; one whole-function
integration test for `restore_after_exit`; and (optionally)
splitting the move-only commits from the feature commit for
easier git-blame archaeology.

## What I learned

Terminal state restoration is a "fire defensively, log
quietly" problem: the cost of an extra ANSI reset on a
terminal that didn't need it is zero (it's a no-op), but the
cost of *missing* the reset is a parent shell that's
unusable until the user types `reset`. The right shape is to
issue every plausibly-needed sequence at exit, not just the
canonically-paired one. The harder lesson is the
`PopKeyboardEnhancementFlags` + `ResetKeyboardEnhancementFlags`
+ `DisableModifyOtherKeys` triad — these are three different
historical layers of "let the app see all keys" protocol
(kitty, kitty-stack-clear, xterm-modifyOtherKeys), and a
modern TUI may have engaged any of them depending on the
terminal it detected. Resetting all three is cheap belt-and-
suspenders; leaving one engaged is a UX bug.
