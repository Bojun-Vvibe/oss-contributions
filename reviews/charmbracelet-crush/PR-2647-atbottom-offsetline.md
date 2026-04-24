# PR #2647 — fix(ui): AtBottom() early exit not accounting for offsetLine

**Repo:** charmbracelet/crush  
**Surface:** `internal/ui/list/list.go` (`AtBottom()` method)  
**Severity:** Low-Medium (UX bug in auto-follow scroll; correct one-liner)

## What changed

In the `AtBottom()` walk, the early-exit guard compared raw
`totalHeight` against `l.height`:

```go
for idx := l.offsetIdx; idx < len(l.items); idx++ {
    if totalHeight > l.height {
        return false
    }
    ...
}
```

The walk starts at `l.offsetIdx`, but `l.offsetLine` represents how many
lines of *that first item* are scrolled out of view. So the effective
visible content is `totalHeight - l.offsetLine`. When the offset item is
tall and partially scrolled out, the old guard would wrongly bail out
saying "not at bottom" even though the visually rendered content fits.

Fix:

```go
if totalHeight-l.offsetLine > l.height {
    return false
}
```

Plus a clarifying comment.

## What's risky / wrong / missing

1. **Off-by-one risk on equality.** The condition is `>`, not `>=`.
   `totalHeight - l.offsetLine == l.height` therefore continues the
   walk and falls through to whatever return is at the bottom (not
   shown in the diff window). Confirm that the post-loop return is
   `true` for the exact-fit case — otherwise the fix introduces a
   one-line edge case where a perfectly-sized list reports
   `!AtBottom()`.

2. **`l.offsetLine` should be ≤ height of `items[offsetIdx]`** by
   construction. If invariants ever drift (e.g. a future code path
   leaves `offsetLine` stale across an item-list mutation),
   `totalHeight - l.offsetLine` could go negative, and `negative >
   l.height` is always false → loop runs to completion → `AtBottom()`
   does the heavy walk every time. That's a correctness no-op but a
   performance cliff. A `debug.Assert(l.offsetLine >= 0 && l.offsetLine
   <= heightOf(items[l.offsetIdx]))` would catch the drift.

3. **No regression test.** This is fixing a known issue (#2481) about
   auto-follow desyncing after manual scroll-back. A test that
   constructs items where the offset item is tall and partially
   scrolled, then asserts `AtBottom()` returns `true`, would lock in
   the fix. The PR is +3/-1 with no test addition.

4. **Comment is good but slightly imprecise.** "the number of lines of
   the item at offsetIdx that are scrolled out of view" — clarify
   whether this counts wrapped lines or logical lines, since the
   `list.go` model conflates the two in a few other places.

## Suggested fix

Land as-is for the regression unblock, but follow up with:
- A unit test in `internal/ui/list/list_test.go` that wires up a fake
  item list with a tall offset item and asserts both `AtBottom() ==
  true` for the fix case and `AtBottom() == false` for a clearly
  not-at-bottom case.
- An assertion (debug build) on `offsetLine` invariants.

## Severity

Low-Medium. UX-visible (auto-follow randomly stops working) but
the change itself is minimal and direction-correct. A reviewer who
already approved is reasonable; the test-coverage gap is the main
critique here.
