# Review: charmbracelet/crush #2647 — fix(ui): fix AtBottom() early exit not accounting for offsetLine

- **Repo**: charmbracelet/crush
- **PR**: #2647
- **Head SHA**: `4e8269e7a4f0aa75629d42d140c094d9869951ed`
- **Author**: octo-patch

## What it does

Two-line fix (plus a clarifying comment) for `internal/ui/list/list.go`'s
`AtBottom()` method. Fixes #2481: auto-follow mode failed to re-engage
after incrementally scrolling back to the bottom (e.g. via trackpad).

## Diff notes

- `internal/ui/list/list.go:88` — early-exit guard inside the
  `for idx := l.offsetIdx; ...` loop changed from
  `if totalHeight > l.height` to
  `if totalHeight-l.offsetLine > l.height`. This now matches the
  invariant used by the function's *final* return (per the PR
  description), making the two checks consistent.
- `internal/ui/list/list.go:85-86` — new two-line comment that
  documents what `l.offsetLine` actually represents ("number of lines
  of the item at offsetIdx that are scrolled out of view"). This
  context was missing and is the real reason the bug existed —
  someone in the future was always going to write
  `totalHeight > l.height` and not realize it was wrong.

## Concerns

1. **No test added.** The bug is small but the fix is in a function
   (`AtBottom()`) that gates auto-follow behavior — a regression
   would be very visible to users (auto-follow stops working) but
   easy to miss in code review. Bubble Tea / Lip Gloss list logic is
   testable without a real TTY. A unit test that constructs a `List`
   with `offsetLine > 0` and asserts `AtBottom()` returns true at the
   boundary (`totalHeight == l.height + l.offsetLine`) would be
   ~10 lines and would catch the next regression.
2. The PR description says the fix was verified by "open a long
   session, scroll up, scroll back down incrementally". That's the
   right manual test, but it depends on the reviewer being able to
   reproduce a session long enough that `offsetLine > 0` happens.
   Worth asking the author to capture the failing case in a test
   fixture.
3. The change is genuinely two characters of behavior (`-l.offsetLine`),
   has a clear story, and the new comment makes the invariant
   explicit. Risk surface ≈ zero — the only way this introduces a
   regression is if the *final* return statement (which already uses
   the same condition) was itself wrong, which the PR doesn't claim.

## Verdict

merge-after-nits
