# charmbracelet/crush PR #2774 — chore(ui): hypercrush small type treatment

- **PR:** https://github.com/charmbracelet/crush/pull/2774
- **Author:** meowgorithm
- **Head SHA:** `ce673448` (full: `ce673448e4f3ca03b842f0b5fb16e9f29368402a`)
- **State:** MERGED
- **Files touched:**
  - `internal/ui/common/elements.go` (+3 / -3)
  - `internal/ui/logo/logo.go` (+12 / -4)
  - `internal/ui/model/header.go` (+19 / -2)
  - `internal/ui/model/sidebar.go` (+3 / -1)
  - `internal/ui/model/ui.go` (+1 / -0)
  - `internal/ui/styles/quickstyle.go` (+1 / -0)
  - `internal/ui/styles/styles.go` (+1 / -0)

## Verdict

**merge-after-nits**

## Specific refs

- `internal/ui/common/elements.go:84` — `formatCredits` renamed to `FormatCredits` so the header can call it. Capitalizing for cross-package use is correct, but the docstring above it (line 88) was updated to match. No other call sites in the diff — verified `hcInfo` uses the new name.
- `internal/ui/logo/logo.go:148-160` (`SmallRender` signature) — now takes `o Opts` and branches on `o.Hyper`: `name = "HYPERCRUSH"` and `charm = "Charm™"` (no leading space) for hyper, otherwise prepends a space to `Charm™`. Subtle behavior: the leading-space toggle compensates for "HYPERCRUSH" being wider than "Crush"; that's an alignment trick that should be commented because future me will absolutely "fix" it.
- `internal/ui/model/header.go:139-156` — `drawHeader`/`renderHeaderDetails` grow a `hyperCredits *int` parameter. The render guard `if com.IsHyper() && hyperCredits != nil` is correct; the only thing missing is a test confirming non-hyper providers don't accidentally render a credits glyph if a stale `hyperCredits` pointer leaks through.

## Rationale

Pure UI/branding diff for the Hyper provider's "HYPERCRUSH" treatment. The substantive code change is plumbing `Opts.Hyper` and `hyperCredits *int` through `SmallRender`/`drawHeader`/`renderHeaderDetails`, which is mechanical and correct. The two nits worth raising before merge: (1) the leading-space alignment trick in `logo.go:154-156` deserves a one-line comment so a future contributor doesn't normalize it away; (2) the renamed exported `FormatCredits` belongs in `common/elements.go` doc-wise, but it's really a string-formatting helper — would fit better in a `formatting.go` or `strings.go` next to `formatTokensAndCost`. Neither blocks the merge; both are cleanup-on-touch material.

