# charmbracelet/crush #2772 — fix: group header hidden on wrap-to-first in model selector

- **Repo:** charmbracelet/crush
- **PR:** https://github.com/charmbracelet/crush/pull/2772
- **HEAD SHA:** `8e918add67be4869808afb8d3d72adf1f4524803`
- **Author:** mkaaad
- **Verdict:** `merge-as-is`

## What the diff does

One-line fix at `internal/ui/dialog/models.go:182` — adds
`m.list.ScrollToTop()` after `m.list.SelectFirst()` in the
"wrap-from-last-to-first" branch of the model-selector key handler.

Prior code (still visible at `:179-186`):

```go
if m.list.IsSelectedLast() {
    m.list.SelectFirst()
    // missing ScrollToTop()
} else {
    m.list.SelectNext()
}
```

The bug: pressing Down on the last visible item correctly wraps the
*selection cursor* back to the first item, but the underlying scroll
viewport does not reset. If the user had scrolled the list far enough
that the topmost group header was no longer in the visible window,
the wrap-to-first leaves the selection on item-0 logically but the
viewport still rendered without item-0's group header at the top.
Result: the first group's header was invisible, breaking the visual
hierarchy that distinguishes (e.g.) "Anthropic" / "OpenAI" /
"Google" model groups.

The fix calls `ScrollToTop()` so the viewport offset is re-anchored to
the top, which puts both the selected item and its group header back
on screen.

## Why it's right

The fix is at the exact site of the bug (the wrap branch already calls
`SelectFirst()`; there's no other branch where the viewport could be
left non-zero with the selection at index 0 in this code path). The
asymmetry with the `else` branch (`SelectNext()` does not need a
viewport reset because it advances by one and the list view tracks
the cursor) is correct: only the wrap case crosses the viewport
window in a single keystroke.

`SelectFirst()` and `ScrollToTop()` are the two independent
responsibilities — selection state and viewport state — and the bug
was that wrap was updating only the first. Same idiom is presumably
applied symmetrically on the wrap-from-first-to-last path; worth
confirming (see nit 1).

The change is one line, minimum diff, no new tests required for a
visual-only viewport-anchor fix that's hard to assert programmatically
without snapshot testing the rendered list.

## Nits

1. **Symmetric wrap-from-first-to-last path** at the Up handler
   (not in this diff) likely needs the parallel `ScrollToBottom()`
   call after `SelectLast()`. Worth confirming in this PR or a
   follow-up — if the bug exists going forward, it almost
   certainly exists going backward.

2. **No test.** `bubbletea` lists can be unit-tested by constructing
   the list, sending a `tea.KeyDown` until the wrap fires, then
   asserting `list.Index() == 0` and `list.View()` contains the
   first group header substring. That said, the bug is genuinely
   one-line and the assertion shape is brittle to bubbles
   internals; a snapshot-style test that captures the rendered
   view before/after wrap would be more robust but is also more
   maintenance.

3. **Idempotency**: `ScrollToTop()` after `SelectFirst()` is safe
   even when the user wraps from a position already at the top
   (selection was on the last item, list is short enough that
   the top was already visible) — it's a no-op in that case.
   Worth a one-line comment naming the bug the line fixes
   (`// reset viewport so first group's header is visible after wrap`)
   so a future reader doesn't strip it as redundant.

## Verdict rationale

One-line surgical fix at the exact wrap site, right semantics
(selection and viewport are independent and both need to advance to
the top on wrap), no risk of regression on the non-wrap path. Worth
checking the symmetric Up path and adding a one-line comment, but
neither is blocking.

`merge-as-is`
