---
pr: 19625
repo: openai/codex
sha: 45b3851384fb
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex #19625 — Reset TUI keyboard reporting on exit

- **URL**: https://github.com/openai/codex/pull/19625
- **Author**: etraut-openai
- **Head SHA**: 45b3851384fb
- **Size**: +328/-195 across 3 files (`codex-rs/tui/src/lib.rs`, `codex-rs/tui/src/tui.rs`, `codex-rs/tui/src/tui/keyboard_modes.rs` (new))

## Scope

Two-part change:

1. **Refactor**: extracts ~150 lines of keyboard-enhancement detection logic (`DISABLE_KEYBOARD_ENHANCEMENT_ENV_VAR`, `parse_bool_env`, `running_in_wsl`, `running_in_vscode_terminal`, `keyboard_enhancement_disabled_for`, Windows `cmd.exe` `TERM_PROGRAM` probe) out of `tui.rs` into a new dedicated `tui/keyboard_modes.rs` module with the same test surface.
2. **Behavior fix**: introduces a new `restore_after_exit()` exit path replacing the old `restore()` (`lib.rs:1586`, `lib.rs:1605`) that explicitly resets the kitty keyboard protocol mode rather than relying on `PopKeyboardEnhancementFlags` from a possibly-corrupted stack. Adds a `KeyboardRestore::ResetAfterExit` variant alongside `PopStack` (`tui.rs:278-279`).

## Specific findings

- **Root cause is correctly diagnosed.** The kitty keyboard protocol stack is process-lifetime state inside the terminal emulator, not the TUI process. If codex panics, is SIGKILLed, or exits between `Push` and `Pop`, the user's terminal is left in a state where Vim's Esc/arrow-keys break, shell ctrl-key bindings misbehave, and the only fix is `reset(1)` or restart-the-terminal. `PopKeyboardEnhancementFlags` doesn't help because (a) the stack might already be empty, (b) the enabled flags might not match what was pushed if codex crashed mid-frame, (c) some terminals' `Pop` is best-effort. Sending an unconditional "set flags to 0" CSI escape on exit is the correct fix.

- **Refactor and behavior change in one PR is acceptable here** because the refactor is a strict module-extraction with the same public surface and no logic delta — `vscode_terminal_detected`, `term_program_is_vscode`, `keyboard_enhancement_disabled_for`, and the Windows `cmd.exe` probe all move byte-identical (per the diff). Reviewer can verify with `git log --follow` on each function's behavior; if anything changed mid-move, that's the spot to push back. The behavior change is genuinely localized to the new `restore_after_exit` + `ResetAfterExit` variant.

- **Test deletions**: `keyboard_enhancement_env_flag_parses_common_values`, `keyboard_enhancement_auto_disables_for_vscode_in_wsl`, `keyboard_enhancement_auto_disable_requires_wsl_and_vscode` are all removed from `lib.rs`. Reviewer must confirm they're recreated in `keyboard_modes.rs` with identical assertions; if they're dropped on the floor, the WSL+vscode auto-disable matrix loses regression coverage. (`keyboard_modes.rs` has 469+ lines of new code so the room is there — visual confirmation needed.)

- **`restore()` → `restore_after_exit()` renaming is good** because it makes the "this is the terminal-shutdown path, not the suspend path" intent explicit at the callsite. Job-control suspend (`SIGTSTP`) wants `PopKeyboardEnhancementFlags`-style cleanup so resume can re-push; full exit wants the unconditional reset. The two `KeyboardRestore` variants (`PopStack` vs `ResetAfterExit` at `tui.rs:278-279`) encode this distinction in the type, which is the right shape.

- **`if self.active` guard at `lib.rs:1605` in `TerminalRestoreGuard::restore`** is preserved correctly — the guard only fires once. But note: if `restore_after_exit()` returns `Err`, `self.active = false` does not get set. That means a second drop-time call would re-enter `restore_after_exit`. Probably fine (the operation is idempotent), but consider whether to set `active = false` unconditionally before the call (the goal is "we tried", not "we succeeded").

- **`let _ = restore_after_exit(); // ignore any errors as we are already failing`** at `tui.rs:328` is the right pattern for panic-handler cleanup. The error doesn't matter because the user is already seeing a panic message.

- **`enhanced_keys_supported` check at `tui.rs:338`** still gates push on `!keyboard_enhancement_disabled() && supports_keyboard_enhancement()` (per crossterm's runtime probe). Reviewer should confirm `restore_after_exit` is called *unconditionally* on exit even when `enhanced_keys_supported == false` — if codex never enabled the protocol, the reset CSI is a no-op on a sane terminal, and on a buggy one it's defensive cleanup. The visible diff doesn't make this fully clear; spot-check the call sites.

- **Kitty protocol reset CSI is `\x1b[<u`** (push 0) or `\x1b[=0u` (set to 0) depending on intent. The PR comment doesn't say which sequence is being emitted — worth a doc-comment in `keyboard_modes.rs` next to the implementation explaining the choice and linking to https://sw.kovidgoyal.net/kitty/keyboard-protocol/.

- **No test for the actual fix.** The new `restore_after_exit` path has no unit test (it shells out to crossterm escape-sequence emission, which is hard to mock). Acceptable, but a smoke test on Linux with `script(1)` capturing the emitted bytes would be cheap and would catch a future "someone replaced the unconditional reset with a Pop again" regression.

## Risk

Low. The refactor is mechanical, the behavior change is additive (new variant + new fn name + new callsite, old path deleted), and the failure mode of "we now emit one extra CSI on exit even when we didn't push anything" is benign on every terminal codex actually targets. The risk surface is the test-deletion correctness — if the moved tests are not in `keyboard_modes.rs`, the WSL+vscode matrix is uncovered.

## Nits

1. Confirm the three deleted tests (`parse_bool_env`, `keyboard_enhancement_auto_disables_for_vscode_in_wsl`, `keyboard_enhancement_auto_disable_requires_wsl_and_vscode`) are recreated verbatim in `keyboard_modes.rs`.
2. Doc-comment the exact CSI sequence emitted by `restore_after_exit` and link the kitty protocol spec.
3. Set `self.active = false` *before* calling `restore_after_exit()` in `TerminalRestoreGuard::restore` so a re-entry can't re-call.
4. Consider a `script(1)`-driven smoke test that asserts the reset CSI is emitted on clean exit.
5. PR description should explicitly call out the "this fixes Vim/Esc breaking after codex crashes" user-visible symptom — current title "Reset TUI keyboard reporting on exit" is accurate but doesn't sell the fix's value.

## Verdict

**merge-after-nits** — correct root-cause diagnosis, right architectural shape (named variant + dedicated exit path), and a clean module extraction. Test-relocation needs visual confirmation but the structural change is sound.

## What I learned

The kitty keyboard protocol's push/pop stack model is a beautiful API in steady state and a footgun under crashes. Any TUI that pushes flags should treat "reset to known state" as the exit contract, not "pop our push". This is the same pattern as `tcsetattr(TCSANOW, &saved_termios)` in classic Unix TUI shutdown — you save what was there, you restore to that, you don't try to undo your own changes. Codex's `ResetAfterExit` variant captures this well by naming the lifecycle (exit vs suspend) rather than the mechanism (pop vs reset).
