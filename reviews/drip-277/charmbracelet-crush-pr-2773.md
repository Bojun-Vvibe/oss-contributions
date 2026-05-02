# Review — charmbracelet/crush#2773

- PR: https://github.com/charmbracelet/crush/pull/2773
- Title: fix(ui): include cancelled prompts in arrow-up history
- Head SHA: `bafe8f8c414d4a130e770e70a178718cdbf0ec32`
- Size: +9 / −2 across 1 file
- Verdict: **merge-after-nits**

## Summary

Bug fix in `internal/ui/model/ui.go`: when a user cancels an in-flight
agent run (Ctrl-C), the prompt that initiated the run never made it
back into arrow-up prompt history because `sendMessage`'s tea.Cmd
returned `nil` on both the cancellation and normal-completion paths.
The PR introduces a sentinel `agentRunCompleteMsg` that fires in
both terminal cases; the main `Update` loop now triggers
`m.loadPromptHistory()` when it sees that message, so cancelled
prompts become arrow-up-able just like completed ones.

## Evidence

- `internal/ui/model/ui.go:160-163` defines the new
  `agentRunCompleteMsg struct{}` with a comment explaining its dual
  role (normal finish OR cancellation).
- Lines 593-594 add the `case agentRunCompleteMsg:` branch in
  `Update`, which appends `m.loadPromptHistory()` to the command
  batch.
- Lines 3163-3173 in `sendMessage` swap the two `return nil`
  branches for `return agentRunCompleteMsg{}`. The error branch
  (non-cancel) still returns the existing `util.InfoMsg{Type:
  InfoTypeError, ...}`, which is correct — failed runs should not
  pollute prompt history.

## Notes / nits

- `m.loadPromptHistory()` likely re-reads from disk / DB on every
  agent-run completion. For a long session with frequent prompts
  that's fine, but if `loadPromptHistory` is anything more than a
  cheap read it might be worth diffing against the in-memory list
  instead of full reload. The change is consistent with the
  existing `case ... m.promptHistory.index = -1` reset block above
  it, so probably already cheap — worth a quick perf sanity check.
- Naming nit: `agentRunCompleteMsg` reads as "the run completed
  successfully", but the comment correctly notes it also fires on
  cancellation. `agentRunFinishedMsg` or `agentRunEndedMsg` would
  better convey "terminated for any reason." Bikeshed-tier.
- No test in this diff. The cancellation path is UI-state-driven
  and tea-bubble model logic is testable; even one test that drives
  `Update` with a synthetic `agentRunCompleteMsg` and asserts
  `loadPromptHistory` was scheduled would lock in the fix.

Ship after the optional rename / test.
