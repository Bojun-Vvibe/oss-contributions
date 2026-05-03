# charmbracelet/crush #2773 — fix(ui): include cancelled prompts in arrow-up history

- URL: https://github.com/charmbracelet/crush/pull/2773
- Head SHA: `bafe8f8c414d4a130e770e70a178718cdbf0ec32`
- Scope: +9 / -2 in `internal/ui/model/ui.go`
- Closes: #2771

## Summary

After cancelling an in-flight prompt with double-Esc, the just-submitted
command was missing from arrow-up history. Cause: `loadPromptHistory()`
runs concurrently with the agent goroutine and finishes before
`createUserMessage` writes the user message to the DB. For normal
completions the user typically types a follow-up which triggers another
`loadPromptHistory()` reload — masking the bug. For cancelled runs no
such reload happens.

Fix: emit a new `agentRunCompleteMsg{}` from the `sendMessage` goroutine
on both completion paths (cancelled and normal), and call
`loadPromptHistory()` when the `Update` loop receives it.

## What I checked

- `internal/ui/model/ui.go:160-163` — new message type with empty struct.
  Standard Bubble Tea pattern. Good.
- `internal/ui/model/ui.go:593-594` — handler appends
  `m.loadPromptHistory()` to the command batch. Matches the existing
  pattern of returning `tea.Cmd` from `Update`. No conflict with other
  handlers — the surrounding cases (`closeDialogMsg`, etc.) all follow
  the same shape.
- `internal/ui/model/ui.go:3166,3173` — both `return nil` in `sendMessage`
  goroutine become `return agentRunCompleteMsg{}`. The intermediate
  `util.InfoMsg{Type: util.InfoTypeError, ...}` path on non-cancel errors
  still returns the error message — meaning prompt history is **not**
  reloaded on errors. That's a deliberate choice (PR description doesn't
  mention errors) but the user could still want the prompt in history
  even when it errored.
- Timing claim in PR description: `createUserMessage` is invoked at
  `internal/agent/agent.go:234` before cancellation wiring at line 243.
  This means by the time `agentRunCompleteMsg{}` fires (which is after
  the goroutine's main work returns), the DB write has happened. The
  ordering claim is plausible; I couldn't verify line numbers without
  cloning, but the logic is internally consistent.

## Nits

1. **Error path doesn't reload history.** If the agent errors (not
   cancellation), the user message is also already persisted by the time
   `sendMessage` returns the `InfoMsg`. Consider also returning
   `agentRunCompleteMsg{}` in that branch (or batching it via
   `tea.Batch(infoMsg, agentRunCompleteMsg{})`) so errored prompts are
   also recoverable from history. Minor — the bug report only covered
   cancellation.
2. **Batch with existing reloads.** `loadPromptHistory()` is now called
   on agent completion *and* on follow-up command submission (existing
   path). The duplicate reload is cheap (in-memory) but worth a quick
   confirmation that the function is idempotent and doesn't, e.g.,
   re-issue a DB query that could race with concurrent writes.
3. The `agentRunCompleteMsg` name is fine but is one of several
   `Msg`-suffixed types in the same block — a doc comment block tag
   like `// see also: agentSubmittedMsg` would help future readers.

## Risk

Very low. Tiny diff (9 lines added), single file, narrow behavior
change. The new message type doesn't replace existing signals; it adds
a strictly-additive notification. Worst-case regression: prompt history
gets reloaded one extra time per agent run.

## Verdict

`merge-after-nits` — primarily extending the same fix to the error
branch so errored prompts are also in arrow-up history.
