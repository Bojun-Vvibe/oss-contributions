# openai/codex #19917 — Allow /statusline and /title slash commands during active turns

- URL: https://github.com/openai/codex/pull/19917
- Head SHA: `9cd0e96cee4938d08ace8f3a027a8264de4c99eb`
- Files: 1 (`codex-rs/tui/src/slash_command.rs`)
- Size: +5 / −3

## Summary

Moves `SlashCommand::Title` and `SlashCommand::Statusline` from the
"unavailable during task" arm into the default `available_during_task() = false`
fall-through. Both commands mutate purely client-side display state
(window title, statusline content) and don't interact with the running
turn's protocol or model-state, so disabling them while a turn was
in-flight was a UX regression with no underlying correctness reason.

## Specific references

- `codex-rs/tui/src/slash_command.rs:196-200` adds the two variants to the
  fall-through arm (alongside `Mcp`, `Apps`, `Plugins`, `AutoReview`,
  `Feedback`, `Quit`). Alphabetical/grouping ordering is consistent with
  surrounding entries — the two new lines sit between `Plugins` and
  `AutoReview`, which is where future readers will look.
- `:212-213` removes the explicit `SlashCommand::Statusline => false` and
  `SlashCommand::Title => false` arms — same default behavior (returns
  `false` for not-listed-as-true commands), so the diff is a refactor from
  "explicit deny" to "default deny" for the keep-disabled case (`Theme`)
  and a flip-to-default-allow for the two newly-permitted commands.
  Wait — re-reading: the default arm in this match returns `true` for the
  listed variants; `Theme => false` is the only remaining explicit deny.
  The two removed `=> false` arms are *promoted* to the implicit-allow
  default. Net effect: `Title` and `Statusline` now return `true` from
  `available_during_task()`.
- Test rename at `:251-254`: `goal_command_is_available_during_task` →
  `certain_commands_are_available_during_task`, extending it with two
  new asserts for `Title` and `Statusline`. The rename is correct because
  the test's invariant is now "the set of during-task-available commands
  includes these three" not "Goal specifically".

## Risk

Almost zero. Both commands are pure-display mutations:
- `/title` updates the terminal window title via OSC 0/2 sequences (host
  terminal capability, no model interaction).
- `/statusline` updates a TUI-rendered status string (purely local
  React-style re-render).

Neither command:
- sends a message to the model
- mutates the running turn's tool-call queue
- modifies session state that the in-flight turn reads from

So enabling them mid-turn cannot interfere with the turn's correctness.

## Nits (non-blocking)

- The PR conflates two refactors: the variant-list expansion at `:196-200`
  and the explicit-arm cleanup at `:212-213`. They're coupled (you can't
  do one without breaking the match exhaustiveness), so this is fine, but
  a one-line PR description would help future archeology — at first glance
  the diff looks like it's doing more than it is.
- `Theme` is the lone surviving `=> false` arm. If `Theme` is also a pure
  display mutation (which it should be — theming = re-render with new
  palette), the same argument applies and a follow-up could promote it
  too. Worth a sentence in the PR explaining why `Theme` was deliberately
  excluded (likely: theme switch causes a full TUI re-layout that races
  against streaming output).
- No negative test pins that `Theme` is *still* unavailable during a task,
  so a future "let's allow everything during a task" sweep that flips
  `Theme => false` to default-allow would pass CI silently.

## Verdict

`merge-as-is` — minimal, correct, has positive test coverage for the new
behavior. The `Theme` exclusion question is a comment-worthy clarification,
not a blocker.
