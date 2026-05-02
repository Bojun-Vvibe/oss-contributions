# charmbracelet/crush #2773 — fix(ui): include cancelled prompts in arrow-up history

- **Repo:** charmbracelet/crush
- **PR:** #2773
- **URL:** https://github.com/charmbracelet/crush/pull/2773
- **Head SHA:** `bafe8f8c414d4a130e770e70a178718cdbf0ec32`
- **Files touched:** `internal/ui/model/ui.go` (+9 -2)
- **Verdict:** merge-after-nits

## Summary

Quality-of-life fix for the TUI prompt history. When the user submitted
a prompt and then `Ctrl+C`'d the in-flight agent run, the cancelled
prompt was not appearing in the arrow-up history because the prompt
history was only refreshed on a successful agent completion. The fix
introduces a new `agentRunCompleteMsg` Bubble Tea message emitted on
*both* normal completion and `context.Canceled`, which the Update loop
consumes by calling `m.loadPromptHistory()`.

## Specific notes

- **`internal/ui/model/ui.go:160-162`** — new message type with a
  doc-comment explaining the dual-purpose (normal vs. cancellation).
  Clean.
- **Lines 590-592** — `case agentRunCompleteMsg:` simply appends
  `m.loadPromptHistory()` to `cmds`. No state changes beyond the
  history reload, which is the right minimum.
- **Lines 3163-3173** — the `sendMessage` callback that previously
  returned `nil` on both cancel and success now returns
  `agentRunCompleteMsg{}`. Symmetric, easy to reason about. The error
  path still returns `util.InfoMsg{Type: InfoTypeError, ...}` so
  errored prompts won't trigger a history reload — that's a deliberate
  choice (errored prompts don't enter history yet, presumably because
  `loadPromptHistory` reads from a persistence layer that records
  prompts only after successful enqueue/cancellation).

## Nits

- If `m.loadPromptHistory()` is even mildly expensive (DB read, file
  parse), firing it on every completion may become a hotspot for
  rapid-fire short prompts. A debounce or "only reload if
  lastSubmittedAt > lastLoadedAt" guard would future-proof this.
- The doc comment says "agent run finishes (normally or via
  cancellation)" but doesn't mention the *error* path; brief addition
  would clarify why errors don't fire this message.
