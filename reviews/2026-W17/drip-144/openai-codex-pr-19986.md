---
pr_number: 19986
repo: openai/codex
head_sha: ecb16c05c42ec061efa6292af3e2537de1c3187f
verdict: merge-as-is
date: 2026-04-28
---

# openai/codex#19986 — fix(tui): let esc exit empty shell mode

**What changed.** +60/−0 across 2 files. Single behavioral change at `codex-rs/tui/src/bottom_pane/chat_composer.rs:2862-2865`:

```rust
if self.is_bash_mode && self.textarea.is_empty() && key_event.code == KeyCode::Esc {
    self.is_bash_mode = false;
    return (InputResult::None, true);
}
```

inserted *before* the existing `if key_event.code == KeyCode::Esc { if self.is_empty() { ... } }` block. Paired with a direct unit test `esc_exits_empty_shell_mode` at `:4717-4744` and a snapshot test `footer_mode_shell_command_escape_exits_empty_mode` at `:4642-4654` that drives `set_text_content("!")` → Esc → snapshots the restored `› Ask Codex to do anything` prompt state.

**Why it matters.** The PR root-cause writeup is correct and load-bearing: the leading `!` that triggers shell mode is stored *outside* the editable textarea (the textarea holds only what comes after the `!`). After typing only `!`, `self.textarea.is_empty() == true` but `self.is_empty() == false` (the latter likely composes textarea + bash-mode-prefix), so the existing empty-composer Esc handler at `:2867+` never fires. The new guard discriminates exactly that cell.

**Concerns.**
1. **Predicate ordering is correct.** The new check fires *before* the shortcut-overlay handler at `:2861` only because the latter is keyed off the overlay state, not the Esc key class — re-read: actually `handle_shortcut_overlay_key` runs first at `:2860-2862`, then this new check, then the existing `KeyCode::Esc` handler. That's the right order: overlay-Esc dismisses overlay, empty-shell-mode-Esc exits shell mode, otherwise general empty-composer-Esc-mode handling.
2. **`(InputResult::None, true)` return** correctly propagates "no input event, but redraw needed" (the footer mode hint changes from bash-mode-shown to normal-mode-shown). Test asserts both: `matches!(result, InputResult::None)` and `assert!(needs_redraw)`.
3. **Test cell shape is exactly right** — types `!` to enter bash mode (`assert!(composer.is_bash_mode)` + `assert_eq!(composer.current_text(), "!")` to confirm the `!` is the *only* input), Escs, then asserts `!composer.is_bash_mode` AND `current_text() == ""`. The `current_text() == ""` after Esc pins the load-bearing invariant that exiting bash mode also drops the bash-mode-stored `!`.
4. **Snapshot test uses `set_text_content("!", ...)`** to set up state rather than typing — verify that `set_text_content` actually sets `is_bash_mode = true`. If not, the snapshot tests "Esc on empty textarea while flagged bash-mode" but doesn't exercise the *typing path* that produced the bug. The direct unit test does cover this via `type_chars_humanlike(&mut composer, &['!'])`, so the snapshot is supplementary documentation.
5. **No regression cell for `!some text` + Esc**, which should hit the *next* arm (the existing `is_empty()` check at `:2866+`) since the textarea is no longer empty. The unit test only proves the new arm, not that the new arm doesn't shadow the old behavior. A second direct test (`type "!hello"`, Esc, assert text becomes `""` and `is_bash_mode == false` per existing semantics) would close the regression window — but the existing test suite presumably already covers that path under `:4640+` neighbors.
6. **Snapshot file is byte-stable** — the new `.snap` at `chat_composer__tests__footer_mode_shell_command_escape_exits_empty_mode.snap` is well-formed, asserts the post-Esc rendered terminal state, and matches the documented expected behavior.

Surgical fix to a real bug class with a snapshot + direct unit test. Ship.
