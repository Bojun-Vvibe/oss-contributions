# charmbracelet/crush PR #2772 — fix: group header hidden on wrap-to-first after scrolling to bottom in model selector

- PR: https://github.com/charmbracelet/crush/pull/2772
- Head SHA: `8e918add67be4869808afb8d3d72adf1f4524803`
- Author: @mkaaad
- Size: +1 / -0

## Summary

In the model-selector dialog, pressing ↓ on the last model wraps selection to the first model — but the first group header (provider title) was being clipped because `offsetIdx` from the prior bottom-of-list scroll position never reset. The wrap logic landed `selectedIdx=1` (skipping the `GroupHeader` at index 0), and `ScrollToSelected` saw "selected is already in the visible window" and did nothing, so rendering started at item 1 and the header at item 0 fell off the top of the viewport.

Fix is one line: call `m.list.ScrollToTop()` immediately after `m.list.SelectFirst()` so `offsetIdx` is forced back to 0 before the next render.

## Specific references from the diff

- `internal/ui/dialog/models.go:182` — adds `m.list.ScrollToTop()` inside the `if m.list.IsSelectedLast() { m.list.SelectFirst(); ... }` branch. The opposite branch (`m.list.SelectNext()`) is unchanged because non-wrap navigation never produces this offset/selected mismatch.

## Verdict: `merge-as-is`

One-line fix, root cause is correctly diagnosed in the PR body, the only place that triggers the wrap is the branch being patched, and there's no risk of regressing the non-wrap path. The fix order — `SelectFirst` then `ScrollToTop` — is also correct: `SelectFirst` resolves the new `selectedIdx`, then `ScrollToTop` resets the offset so the subsequent `ScrollToSelected` (called by the list internals) is a no-op rather than a fight.

## Nits / concerns (post-merge polish)

1. **Symmetric bug for ↑-on-first?** If pressing ↑ on the first item wraps to the last, does the same offset/selected mismatch happen in reverse (last group header rendered with no leading items above it)? Worth a 30-second manual check. If yes, a follow-up PR should add the symmetric `ScrollToBottom()` call in the wrap-to-last branch.
2. **No test.** This dialog is interactive and the list widget is a `bubbles/list` derivative, so a TUI snapshot test is a lift. Not a blocker; the change is provably small. But a regression test that mocks `IsSelectedLast() → true` and asserts `ScrollToTop` is called would prevent reintroduction.
3. **PR description has a typo** (`SelectFirst()` is described as setting `selectedIdx=0` then "looping skipping non-`*ModelItem`"). The asterisk-quoting renders weirdly on GitHub. Cosmetic; the diagnosis is right.
