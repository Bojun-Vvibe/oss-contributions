# charmbracelet/crush #2725 — fix(ui): notification width and text truncation

- Author: meowgorithm (Christian Rocha)
- Head SHA: `c628dce360ff62840f32ad6d73c514c60f0ec253`
- Single file: `internal/ui/model/status.go` (+6 / −4)

## Specifics

- The old code at `status.go:104-108` computed `msgWidth := lipgloss.Width(msg)` first, then truncated with `area.Dx()-indWidth-msgWidth` as the budget — but `msgWidth` is the *current* message width, not the *available* width, so the truncation budget is wrong whenever `msgWidth > area.Dx()-indWidth` (the budget went negative, effectively asking `ansi.Truncate` to truncate to a negative width). Padding then used `max(0, area.Dx()-indWidth-msgWidth)`, which clamped to 0 and produced an unpadded, possibly-overrun status line.
- The new code at `status.go:104-110` correctly computes `avail := max(0, area.Dx()-indWidth-msgPad)` first (where `msgPad := msgStyle.GetPaddingLeft() + msgStyle.GetPaddingRight()` accounts for the lipgloss style's horizontal padding — a previously-ignored term that would have caused 1-2 cell overruns on any styled message), truncates to `avail`, then re-measures via `lipgloss.Width(msg)` and pads to `avail - w`. Two correctness wins: (1) truncation budget is now the available width not the current width, (2) lipgloss style padding is no longer double-counted.
- The post-truncation re-measure is necessary because `ansi.Truncate` may produce a string narrower than `avail` when it lands on a multi-cell rune boundary; the old code didn't re-measure either, so this also fixes a subtle off-by-one padding gap on east-asian-wide-char status messages.

## Concerns

- No regression test added. `internal/ui/model/status_test.go` (if it exists) would be the natural home for a width-vs-padding pinning test; even a table test with `(area.Dx, indWidth, msgPad, msg)` → `(want_visible_width, want_truncation_marker_present)` would lock this down.
- `msgStyle` is captured by reference in `Status.Draw`; if anything mutates its padding between calls the assumption that `msgPad` is stable for the lifetime of the truncation breaks. Reading the surrounding file would clarify whether `msgStyle` is package-level or per-instance.

## Verdict

`merge-as-is` — proper width-math fix from the project lead. The padding-aware budget computation is the kind of detail that only gets caught by someone who has stared at lipgloss layout bugs before; reasoning is sound.

