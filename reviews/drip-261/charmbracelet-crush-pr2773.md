# charmbracelet/crush PR #2773 — fix(ui): include cancelled prompts in arrow-up history

- **Head SHA:** `bafe8f8c414d4a130e770e70a178718cdbf0ec32`
- **Files:** 1 (`internal/ui/model/ui.go`)
- **LOC:** +9 / −2

## Observations

- `ui.go:160-162` — adds a new typed message `agentRunCompleteMsg struct{}` that fires when an agent run finishes for *any* reason (normal completion or cancellation). Tiny, well-commented type — fits the BubbleTea idiom of one msg per event class.
- `ui.go:593-594` — the new case `case agentRunCompleteMsg: cmds = append(cmds, m.loadPromptHistory())` reloads the prompt history. This is the correct fix: the previous flow returned `nil` from the cancellation branch (`ui.go:3166`, old line) which meant `loadPromptHistory()` was never invoked after a cancel, so an arrow-up did not include the prompt the user just cancelled.
- `ui.go:3163-3173` — both the `errors.Is(err, context.Canceled)` branch and the success branch now return `agentRunCompleteMsg{}` instead of `nil`. Symmetric, and the InfoMsg error branch is unchanged so error behavior is preserved.
- One concern: `agentRunCompleteMsg` is dispatched on both cancel and success, which means `loadPromptHistory()` runs twice for a normal turn (once on the existing success path and once on this new path) unless the existing success path was previously firing through a different msg type. Worth a quick check that the success-path reload is not duplicated. Looking at the diff in isolation, the old success path returned `nil` — so this PR is also adding history reload to the *success* path, not just the cancel path. The PR title only mentions cancellation, but the diff is broader. Not necessarily a bug — just file the wider behavior in the PR description.
- No tests added; the file appears to have no existing test coverage for this code path, so this matches local convention. A small unit test mocking `m.loadPromptHistory` would be cheap insurance though.

## Verdict: `merge-after-nits`

- Update the PR description to note that history reload now also fires on success (not only on cancel).
- Confirm `loadPromptHistory()` is not double-invoked on the normal completion path.
