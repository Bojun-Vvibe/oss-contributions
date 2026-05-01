# charmbracelet/crush #2773 — fix(ui): include cancelled prompts in arrow-up history

- **Repo:** charmbracelet/crush
- **PR:** https://github.com/charmbracelet/crush/pull/2773
- **HEAD SHA:** `bafe8f8`
- **Author:** pragneshbagary
- **Verdict:** `merge-as-is`

## What the diff does

Tiny, targeted fix at `internal/ui/model/ui.go`. The shape:

1. Defines a new internal message type `agentRunCompleteMsg struct{}` (`:160-163`) with the comment "sent when an agent run finishes (normally or via cancellation) so that prompt history can be refreshed."
2. Adds a `case agentRunCompleteMsg:` arm in `Update` (`:593-594`) that appends `m.loadPromptHistory()` to the command batch.
3. Replaces the two `return nil` sites at the end of the `sendMessage` async closure (`:3166`, `:3173`) with `return agentRunCompleteMsg{}` — one in the `errors.Is(err, context.Canceled)` branch, one on the success path.

Net result: when the agent run terminates *for any reason other than a non-cancel error*, the prompt is reloaded so the user-typed text becomes addressable via arrow-up. Before the fix, only the success path reloaded history (presumably elsewhere), and a cancelled prompt — the exact case where the user is most likely to want it back, because they pressed Ctrl+C or Esc to redo it — was silently dropped from the recallable set.

## Why it's right

- **Diagnosis is exact.** The bug shape "history reloads only on success" is a classic asymmetry-of-completion-paths bug. The two return sites in `sendMessage` were the obvious candidates; both now route through the same message.
- **The `agentRunCompleteMsg` indirection is the right primitive.** The closure can't call `m.loadPromptHistory()` directly because `m` may have been mutated by the time the closure resolves and the Bubble Tea update loop owns the model state. Returning a `tea.Msg` is the canonical way to ask "please run this on the next Update tick with the current model state." Calling `loadPromptHistory` inline would race the model.
- **Non-cancel errors still return `util.InfoMsg`** (`:3168-3171`) — correctly preserved, because in the error case the user can see the error in the surface and the unsent prompt is *still in the input buffer* (it hasn't been cleared), so it's already recallable; reloading history would be redundant.
- **`promptHistory.draft = ""` and `promptHistory.index = -1`** reset elsewhere in `Update` (visible at `:589-591` context) means the history pointer is in a clean state when the next reload completes — no off-by-one when the user immediately presses arrow-up.
- The change is **17 lines, 4 hunks, all in one file**, with the smallest possible surface for the bug it fixes.

## Nits

- (Optional) The `agentRunCompleteMsg` could carry the cancellation reason (`Cancelled` vs `Completed`) for future telemetry, but that's premature for this fix.
- No automated test, but Bubble Tea test surfaces are notoriously awkward for "an arrow-key press finds the right entry in history after a context.Canceled flow." The fix is small enough and the comment is clear enough that the manual-test cost matches the fix cost.
- The comment "sent when an agent run finishes (normally or via cancellation)" doesn't mention the explicit decision to NOT send it on non-cancel errors. One sentence ("non-cancel errors are surfaced via InfoMsg and the unsent prompt remains in the input buffer, so a reload is unnecessary there") would prevent a future "fix" that adds the message to the error path too and breaks the input-buffer-preservation assumption.

## Verdict rationale

Right diagnosis (asymmetric completion paths), right load-bearing primitive (`tea.Msg` indirection rather than inline call so the Bubble Tea loop owns the state mutation), correct preservation of the error path's distinct semantics (input buffer kept, no history reload), 17-line surface with a single-file blast radius, named comment on the new message type. No nits big enough to block merge.

`merge-as-is`.
