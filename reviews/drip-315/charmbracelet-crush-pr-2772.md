# charmbracelet/crush PR #2772 — fix: group header hidden on wrap-to-first after scrolling to bottom in model selector

- Link: https://github.com/charmbracelet/crush/pull/2772
- Head SHA: `8e918add67be4869808afb8d3d72adf1f4524803`
- Author: mkaaad
- Size: +1 / −0 (1 file)

## Files changed
- `internal/ui/dialog/models.go` — `+1` (one new line: `m.list.ScrollToTop()`)

## Reasoning

This is a one-line UI fix. The PR author has done the legwork: the description includes a clear root-cause walkthrough (item indices, the `SelectFirst` → `ScrollToSelected` chain, and why the `offsetIdx` doesn't reset).

The actual change at `internal/ui/dialog/models.go:181`:

```go
m.list.Focus()
if m.list.IsSelectedLast() {
    m.list.SelectFirst()
    m.list.ScrollToTop()
} else {
    m.list.SelectNext()
}
```

The added `ScrollToTop()` is paired with `SelectFirst()` — i.e. only fires on wrap-around, so it doesn't change behavior of normal `SelectNext()` traversal.

Things I checked in my head:

1. **Does `SelectFirst()` already imply scroll-to-top?** Per the PR's root cause, no — `SelectFirst()` sets `selectedIdx` and `ScrollToSelected()` later only ensures the selected index is *visible*, not that the list is at the top. If the selected index is row 1 (first model, after the GroupHeader at row 0), `ScrollToSelected()` will not necessarily bring row 0 into view — exactly the described bug. The fix is correct.

2. **Symmetry: the up-arrow wrap path.** The PR only patches the down-arrow `IsSelectedLast()` wrap. A symmetric bug presumably exists when pressing ↑ on the first model (wrap to last), where `SelectLast()` is called — but in that case "scroll to last" is implicit because `ScrollToSelected()` will offset the list to show the bottom, including any trailing group header. So no parallel fix is needed. (Worth confirming when this is reviewed by a maintainer who knows the list widget's invariants.)

3. **No test added.** The codebase appears not to have unit tests for this dialog (these are Bubble Tea TUI components driven by `tea.Msg`, hard to assert against). Acceptable for a one-line fix at this scope, but a regression test would be nice if a TUI test harness exists.

## Verdict

`merge-as-is`

One-line, well-explained, low-risk fix. The PR author's root-cause analysis matches the code. Wrap-around-to-first now correctly resets the viewport so the leading group header is visible.
