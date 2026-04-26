---
pr: 2647
repo: charmbracelet/crush
sha: 4e8269e7a4f0aa75629d42d140c094d9869951ed
verdict: merge-as-is
date: 2026-04-27
---

# charmbracelet/crush #2647 — fix(ui): fix AtBottom() early exit not accounting for offsetLine

- **Head SHA**: `4e8269e7a4f0aa75629d42d140c094d9869951ed`
- **Size**: 4-line behavioral change in `internal/ui/list/list.go`

## Summary
Fixes an off-by-`offsetLine` bug in `List.AtBottom()`. The list tracks two scroll quantities: `offsetIdx` (which item is the topmost one in view) and `offsetLine` (how many lines of *that* item are scrolled out the top). `AtBottom()` walks items from `offsetIdx` summing their heights until it exceeds the viewport, and the early-exit condition was comparing raw `totalHeight > l.height` — but the *visible* height is `totalHeight - l.offsetLine`, because the first `offsetLine` lines of the topmost item are off-screen.

## Specific findings
- `list/list.go:85-87` — added comment correctly identifies the semantics: "`l.offsetLine` is the number of lines of the item at offsetIdx that are scrolled out of view, so the effective visible height is totalHeight-l.offsetLine."
- `list/list.go:88` — guard changed from `if totalHeight > l.height` to `if totalHeight-l.offsetLine > l.height`. Algebraically: previously the loop returned `false` (not-at-bottom) when accumulated heights exceeded the viewport, but the comparison was over-counting by `offsetLine`. With the fix, the early exit is invoked at the correct accumulated visible-height boundary.
- The result is that `AtBottom()` will more often return `true` near the actual bottom (not a false negative), which controls auto-scroll behavior. False negatives previously caused the chat list to *not* auto-scroll on new messages when the user was already at the bottom of a long-message session.
- Single-spot fix with no other call-site changes. The `for idx := l.offsetIdx; idx < len(l.items); idx++` loop bounds, `totalHeight += …` body, and downstream return-`true` are all unchanged.
- **No test added.** The list package likely has a `list_test.go`; a 5-line case constructing a List with `offsetIdx=0, offsetLine=3, height=10` and 4 items of heights `[5, 5, 5, 5]` would have caught this and would lock in the fix. Not blocking but recommended for a follow-up.

## Risk
Very low. Tightens an early-exit predicate; the `false` return path is replaced more aggressively with continuing-the-loop, ultimately ending at the same `return true` after `len(l.items)`. The only observable behavior difference is `AtBottom()` returning `true` in cases where it previously returned `false`, which is the bug being fixed.

## Verdict
**merge-as-is** — minimal, correct, well-commented. A regression test would harden it but the fix itself is unambiguous.
