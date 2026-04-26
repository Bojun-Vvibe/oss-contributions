# openai/codex #19618 — Persist shell mode commands in prompt history

- **Repo**: openai/codex
- **PR**: #19618
- **Author**: etraut-openai (Eric Traut)
- **Head SHA**: fcf1e1df5cb2aab194d694476761e208d89335b3
- **Base**: main
- **Size**: +39 / −11 across `chat_composer.rs` (+7/−7), `chatwidget.rs`
  (+19/−2), 1 snapshot, and 2 test files.

## What it changes

Two coordinated changes:

1. **History persistence** (`chatwidget.rs:6115-6125`): introduces
   `submit_shell_command_with_history(command, history_text)` which
   wraps `submit_shell_command` and, if the drain returns
   `QueueDrain::Stop`, also emits `Op::AddToHistory { text: history_text }`.
   Two call sites switch over: the queued-prompt path
   (`chatwidget.rs:6131-6133`, where `history_text = user_message.text`)
   and the direct-input path (`chatwidget.rs:6230`, where
   `history_text = text` including the leading `!`).

2. **Naming refresh** (`chat_composer.rs`): "Bash mode" → "Shell mode"
   in the footer label, doc comment, test name, and snapshot.

## Strengths

- The history insert is gated on `QueueDrain::Stop`, which correctly
  matches "we actually executed the command synchronously". Commands
  that were merely queued (`Continue`) don't yet need a history entry
  because the queue path emits its own. Good invariant.
- `history_text` includes the `!` prefix (line 6230 — `&text`, not
  `&stripped`), which matches what the user typed. This means
  `↑`-recall replays the full command verbatim including the prefix
  and re-enters shell mode automatically. Correct.
- "Bash" → "Shell" rename is right: not all users have bash;
  `$SHELL` may be zsh, fish, etc. The test rename
  (`shell_command_uses_bash_accent_style` →
  `shell_command_uses_shell_accent_style`) keeps the test name
  consistent with the label.
- Snapshot diff updated in the same commit.

## Concerns / asks

- The PR title says "Persist shell mode commands in prompt history"
  but bundles in the Bash→Shell rename, which is a separate concern.
  Two PRs would have been cleaner. Not blocking.
- The new wrapper duplicates `command` and `history_text` parameters
  even though `history_text` is always `format!("!{}", command)` (or
  the original `user_message.text`). For the queued path
  `user_message.text` is preserved; for the direct path
  `text == format!("!{}", stripped)` always holds. A single
  `submit_shell_command_with_history(text)` that strips the `!`
  internally would reduce parameter coupling.
- No assertion that `Op::AddToHistory` is *not* emitted on
  `QueueDrain::Continue` — i.e. that queued-but-not-yet-run commands
  don't double-write history when they later run. The existing test
  expansions in `exec_flow.rs` (+4) and `slash_commands.rs` (+8) may
  cover this; would help to call it out.

## Verdict

`merge-after-nits` — correct fix and a sensible rename, but the
two-PR hygiene point and the parameter-redundancy nit are worth
addressing. The history-on-`Stop`-only invariant is the right
choice and the snapshot+test updates are thorough.
