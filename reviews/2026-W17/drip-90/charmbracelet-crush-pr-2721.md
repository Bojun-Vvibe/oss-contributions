---
pr: 2721
repo: charmbracelet/crush
sha: 024c9d2243f79939eed3830d946484c5cef516a7
verdict: merge-as-is
date: 2026-04-27
---

# charmbracelet/crush #2721 — fix(ui): fix dialog box shift when session rename is active

- **Head SHA**: `024c9d2243f79939eed3830d946484c5cef516a7`
- **Author**: meowgorithm (Christian Rocha, Charm maintainer)
- **Size**: tiny (3-line change in `internal/ui/dialog/sessions_item.go`)

## Summary
Fixes the rename-session dialog item visibly shifting horizontally when entering rename mode. The previous width calculation subtracted `styles.InfoTextFocused.GetHorizontalFrameSize()` (the wrong style for the rename-mode item) and didn't account for the trailing cursor cell, so the input box was 1–2 cells wider than the rendered focused container, pushing it past the right edge.

## Specific findings
- `internal/ui/dialog/sessions_item.go:91-97` — three changes in one block, all correct:
  1. `width - styles.InfoTextFocused.GetHorizontalFrameSize()` → `width - styles.ItemFocused.GetHorizontalFrameSize()`. The `ItemFocused` style is the one actually used to render this row (see the `return styles.ItemFocused.Render(...)` two lines below), so its frame size is the right thing to subtract. The old code was using a sibling style's frame size — a copy-paste-from-the-info-row bug.
  2. `const cursorPadding = 1` — explicit name, scoped tightly. Reserves one cell so the bubbletea `textinput` cursor block doesn't overflow into the next column.
  3. `max(0, ...)` clamp — defends against the case where `width` is smaller than `frameSize+cursorPadding` (very narrow terminal). Without this, `inputWidth` could go negative and the textinput would crash or render junk depending on the bubbles version.
- The fix is local to the rename-mode branch (`if s.focused` inside the rename render path), so non-rename rendering is untouched.
- No test added. The rename UI in `internal/ui/dialog/sessions_item.go` doesn't appear to have width-arithmetic unit tests today (consistent with the file's style), so this is acceptable. A snapshot test would be nice future work.

## Risk
Trivial. Three lines, all defensive, in a pure-rendering function. Worst case: rename input is 1 cell narrower than before, which is the entire point.

## Verdict
**merge-as-is** — author is a maintainer, change is minimal and correct, and the `max(0, ...)` clamp is a strict improvement over the prior unguarded subtraction.
