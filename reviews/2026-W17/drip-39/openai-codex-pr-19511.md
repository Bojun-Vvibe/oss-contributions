# openai/codex #19511 — Keep slash command popup columns stable while scrolling

- **Repo**: openai/codex
- **PR**: [#19511](https://github.com/openai/codex/pull/19511)
- **Head SHA**: `47b317e4b14fed25fdc61afd2bf5067ca3278204`
- **Author**: etraut-openai
- **State**: OPEN (+17 / -4)
- **Verdict**: `merge-as-is`

## Context

When the slash-command popup in the TUI is taller than
`MAX_POPUP_ROWS`, scrolling the visible window changes which
rows are measured for column-width calculations, so command and
description columns visibly jiggle as the user moves up/down.
Fix is to compute column widths over **all** rows, not just the
visible slice.

## Design

A single file change at
`codex-rs/tui/src/tui/src/bottom_pane/command_popup.rs`:

1. **Lines 4-11**: Two new imports —
   `selection_popup_common::ColumnWidthConfig` and
   `ColumnWidthMode`, plus the `_with_col_width_mode` variants of
   `measure_rows_height` and `render_rows`.

2. **Lines 18-22**: New `const COMMAND_COLUMN_WIDTH:
   ColumnWidthConfig` configured with
   `ColumnWidthMode::AutoAllRows` and `name_column_width: None`.
   The `AutoAllRows` mode is the explicit signal that column
   widths should be measured against the full row set, not the
   viewport.

3. **Lines 116-126**: `calculate_required_height` now calls
   `measure_rows_height_with_col_width_mode(...,
   COMMAND_COLUMN_WIDTH)` instead of the plain
   `measure_rows_height`.

4. **Lines 235-247**: `WidgetRef::render_ref` switches to
   `render_rows_with_col_width_mode` and passes the same const.

The fix relies on `selection_popup_common` already exposing a
`_with_col_width_mode` family — this PR is just opting the
command popup into it. The plain `render_rows` /
`measure_rows_height` defaults presumably keep the old
viewport-measured behaviour for callers that want it.

## Risks

- **Cost of measuring all rows** is `O(commands)` instead of
  `O(visible)`, but the slash-command set is small (dozens, not
  thousands) and this is a popup that re-renders on keystrokes,
  not per frame, so this is a non-issue in practice.
- **Width can grow if a long command lands off-screen**, which is
  the intended behaviour but worth flagging: a single
  long-described command will widen the popup permanently while
  the popup is open. That's the right trade vs. jiggling.
- The `name_column_width: None` argument (line 21) is a slightly
  confusing API — passing `None` means "auto", which is what
  `AutoAllRows` already implies. A future cleanup could fold the
  two into one enum, but that's not this PR's job.

## Suggestions

- Worth a TUI screenshot or two-line `before/after` in the PR
  body for reviewers who haven't seen the jiggle.
- Could drop a comment above `COMMAND_COLUMN_WIDTH` explaining
  *why* `AutoAllRows` (i.e., the jiggle) since the const name
  alone doesn't communicate intent. Otherwise readers will have
  to grep `selection_popup_common` to see what the mode does.

## What I learned

This is the canonical "scroll viewport vs. layout viewport"
problem. The fix isn't to remove scrolling — it's to widen the
measurement window past the scroll. `AutoAllRows` is the
TUI-equivalent of CSS `min-width: max-content` on a flex column:
you accept paying for off-screen content so the visible part
stays stable.
