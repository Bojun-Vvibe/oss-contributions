# charmbracelet/crush PR #2773 — fix(ui): include cancelled prompts in arrow-up history

- PR: https://github.com/charmbracelet/crush/pull/2773
- Head SHA: `bafe8f8c414d4a130e770e70a178718cdbf0ec32`
- Author: @pragneshbagary (Hawkeye)
- Fixes: #2771

## Summary

When a user submitted a command and cancelled it before the agent yielded back, the command was missing from arrow-up history for the rest of the session — even though it appeared correctly in chat. Root cause: `loadPromptHistory()` runs concurrently with the `sendMessage` goroutine and finishes before `createUserMessage` writes the user message to the DB; on a normal completion the user types another command which triggers a fresh load that picks up the previous one, but on a cancelled run no such reload is triggered. Fix: introduce `agentRunCompleteMsg{}`, return it from both terminal branches of the `sendMessage` goroutine (cancellation and normal completion), and handle it in `Update` by calling `loadPromptHistory()`. By the time this fires, `createUserMessage` (called at `internal/agent/agent.go:234`, before cancellation wiring at `:243`) has already persisted the message, so the reload reliably picks it up.

## Specific references from the diff

- `internal/ui/model/ui.go:157-163` — declare new message type: `// agentRunCompleteMsg is sent when an agent run finishes (normally or via cancellation) so that prompt history can be refreshed. agentRunCompleteMsg struct{}`.
- `:586-595` — new case in `Update`: `case agentRunCompleteMsg: cmds = append(cmds, m.loadPromptHistory())`.
- `:3156-3173` — both terminal returns in the `sendMessage` goroutine flipped from `return nil` to `return agentRunCompleteMsg{}` (cancellation branch and success branch). The intermediate `util.InfoMsg` error return is left alone.

## Verdict: `merge-as-is`

Surgical 3-spot diff, clear root cause, the fix lives in the layer responsible for the bug (the UI's view of prompt history, not the agent's persistence path), and the existing error-return behavior is preserved. The only reason to nit at all is the lack of a regression test, but this is squarely in TUI-state-machine territory where the project already relies on manual/recorded testing for similar fixes.

## Nits / concerns

1. **Error-path consideration.** When `sendMessage` returns the `util.InfoMsg{Type: InfoTypeError, ...}`, prompt history is *not* refreshed. That's likely correct (the user's command did get persisted by `createUserMessage` *before* the error) — but it means the error path will still drop the cancelled prompt from arrow-up history if the error occurred after `createUserMessage` succeeded. Consider returning `tea.Batch(util.InfoMsg{...}, agentRunCompleteMsg{})` or just routing all three terminal branches through `agentRunCompleteMsg` to keep behavior uniform.
2. **No test.** The PR body uses repro steps but no automated test guards the regression. A small `Update` test injecting a synthetic `agentRunCompleteMsg{}` and asserting that `loadPromptHistory()` is queued would prevent this from silently regressing in a future refactor of the message types.
