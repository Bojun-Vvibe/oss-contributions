# charmbracelet/crush PR #2786 — fix: account for section overhead in sidebar height allocation

- Head SHA: `0e1e099e179cd7d4111d72f376ff62f7f49a0ee7`
- Files changed:
  - `internal/ui/model/sidebar.go` (+5/-1)

## Analysis

Bug fix for sidebar overflow: the four sidebar sections (Files, LSPs, MCPs, Skills) were each allocated items as if a row consumed exactly 1 line, but each section also draws a title line, an internal blank line (from `"\n\n"`), and an inter-section separator. Across four sections the unaccounted overhead totals 11 lines, not the previous magic constant of 6.

Diff at `internal/ui/model/sidebar.go:166-175`:
```
- remainingHeight := remainingHeightArea.Dy() - 6
+ // Account for 4 section titles (1 each), 4 blank lines within sections
+ // (1 each from "\n\n"), and 3 separator lines between sections — total 11
+ // lines of overhead that getDynamicHeightLimits doesn't track.
+ const sectionOverhead = 11
+ remainingHeight := max(0, remainingHeightArea.Dy()-sectionOverhead)
```

Two real improvements beyond the constant bump:

1. The bare subtraction `Dy() - 6` could underflow into a negative `remainingHeight` on very small terminal heights (e.g. someone reproduces by shrinking to ~10 rows). The new `max(0, …)` guard makes the function robust to that, which the old code was not. This is the more important latent fix even ignoring the magic-number change.
2. The named constant + comment turns "what is 6?" into "11 = 4 titles + 4 blanks + 3 separators", which is the kind of arithmetic that drifts every time someone adds a section. A future addition of e.g. a "Hooks" section would now have to update `sectionOverhead` to 14 (5+5+4) and the comment makes the formula explicit.

Risks:
- Brittle to layout changes: if someone changes section rendering to drop the trailing `"\n\n"` or the inter-section separator, this constant has to change in lockstep. A mild improvement would be to derive it from the same constants used by the rendering loop instead of repeating the magic number, but that is a larger refactor and not worth blocking this PR.
- `max` here is the Go 1.21 builtin; project must be on `>=1.21`. The repo's `go.mod` is well above that, so fine.
- Verified via the before/after screen-recordings linked in the PR body; the after-clip clearly fits all sections with no overflow.

## Verdict

`merge-after-nits`

## Nits

- Optional: derive `sectionOverhead` from `len(sectionList)` (titles + blanks) plus `len(sectionList)-1` (separators) so adding a section auto-updates the math. Acceptable as a follow-up.
