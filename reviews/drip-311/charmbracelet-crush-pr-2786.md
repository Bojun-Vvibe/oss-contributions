# charmbracelet/crush #2786 — fix: account for section overhead in sidebar height allocation

- PR: https://github.com/charmbracelet/crush/pull/2786
- Author: mkaaad
- Head SHA: `0e1e099e179cd7d4111d72f376ff62f7f49a0ee7`
- Updated: 2026-05-03T07:36:46Z

## Summary
A 5-line layout fix in `internal/ui/model/sidebar.go:166-173`. Replaces the magic `remainingHeight := remainingHeightArea.Dy() - 6` with a documented `const sectionOverhead = 11` and a `max(0, ...)` clamp to avoid negative heights when the viewport is very small. The old `-6` underestimated the per-section overhead (4 section titles + 4 in-section blank lines + 3 separator lines = 11 lines) so on small terminals the sidebar would overflow.

## Observations
- `internal/ui/model/sidebar.go:166-170`: the comment "Account for 4 section titles (1 each), 4 blank lines within sections (1 each from `\n\n`), and 3 separator lines between sections — total 11 lines of overhead that getDynamicHeightLimits doesn't track" is exactly the kind of explanation that prevents the next contributor from reverting this. Strong inline doc.
- `internal/ui/model/sidebar.go:170`: `max(0, remainingHeightArea.Dy() - sectionOverhead)` — the clamp is necessary because `getDynamicHeightLimits` downstream presumably can't handle a negative argument. Worth confirming the downstream consumer's tolerance for `0`. If `0` causes a divide-by-zero or empty render, the bug just moves; if `0` is a clean "render nothing", we're fine.
- The number `11` is derived from the current sidebar layout (4 sections). If a future PR adds/removes a sidebar section, this constant becomes stale silently — there's no compile-time link between `sectionOverhead` and the actual section list. Two options: (a) compute it from the section list at runtime, or (b) leave a `// IMPORTANT: update if section count changes` comment near the section enumeration. (b) is cheap and probably enough.
- No test added — pure layout math fix in a TUI loop, hard to test without a renderable harness. The conventional `internal/ui/model/sidebar_test.go` (if it exists) could pin the calculation by mocking `remainingHeightArea.Dy()` to known values and asserting `remainingHeight`, but the file is small enough that a manual smoke test is acceptable.
- The PR description should explicitly say which terminal sizes were tested (e.g. "verified at 80x24 and 120x40, sidebar no longer overflows at 80x24"). Without that, a reviewer can't easily reproduce.
- No banned-string risk; no security implications; pure UI layout.

## Verdict
`merge-after-nits`
