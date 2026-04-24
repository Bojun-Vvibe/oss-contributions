# PR-19170 — Skip disabled rows in selection menu numbering and default focus

[openai/codex#19170](https://github.com/openai/codex/pull/19170)

## Context

Refactor in `codex-rs/tui/src/bottom_pane/list_selection_view.rs`. The
`ListSelectionView` (used for popups like microphone picker, plugin
list, model picker) was numbering *visible* rows 1..N including
disabled ones, and would happily default-focus a disabled row if it
happened to be `is_current`. Pressing the digit key for a disabled row
would either select it or no-op silently. The patch:

1. Introduces `Self::item_is_enabled(item)` and replaces ~5 inlined
   copies of `item.disabled_reason.is_none() && !item.is_disabled` with
   a single helper.
2. Adds `first_enabled_visible_idx()` and `enabled_actual_idx(idx)`,
   used in `apply_filter` to *fall back* past disabled rows when the
   previously-selected, current-marked, or initial-selected row is now
   disabled.
3. Adds `actual_idx_for_enabled_number(number)`: digit-key shortcuts
   (`1`–`9`) now index into the *enabled* subset in display order,
   instead of into all visible rows.
4. Renumbers `build_rows()` to assign sequential numbers only to
   enabled rows and pad disabled rows with whitespace of width
   `enabled_row_number_width`.

A regression test
(`disabled_current_rows_skip_default_selection_and_number_shortcuts`)
exercises a 4-row list (`disabled, enabled, disabled, enabled`) and
asserts `selected_actual_idx() == Some(1)` (skips index 0), the rendered
output reads `› 1. Alpha` / `  2. Beta`, and pressing `2` selects
actual index 3. An updated snapshot for the realtime mic picker shows
the visual change: `1. System default` / `Unavailable: Studio Mic
(disabled)` / `2. Built-in Mic` / `3. USB Mic` instead of the previous
`1. / › / 3. / 4.`.

## Why it matters

This is a TUI accessibility bug class. A user pressing `3` to pick "USB
Mic" from a list where `Studio Mic` is shown at index 2 in disabled
state would either not get USB Mic or get nothing, with no feedback.
The fix makes the displayed numbers and the keyboard shortcuts agree.

## Strengths

- Correct call: numbering by the *enabled* subset, not by display
  position, is what users intuitively expect when disabled rows are
  shown grayed-out with explanatory text.
- The `Self::item_is_enabled` helper is the right consolidation. Five
  separate inline copies of the same predicate is exactly the
  refactor-debt that makes future "now also exclude X" changes break
  in three of the five places.
- Padding disabled rows with `enabled_row_number_width + 2` spaces
  preserves column alignment so the description text doesn't jitter as
  you scroll past disabled rows.
- The snapshot diff (`__realtime_microphone_picker_popup.snap`) is the
  best kind of test for this — it locks in the literal terminal output
  including the `›` selection arrow and the renumbering.
- New test `plugin_detail_error_popup_skips_disabled_row_numbering`
  exercises a real upstream surface (the plugin-detail-load-failure
  case) rather than a synthetic minimum.

## Concerns / risks

- The selection precedence in `apply_filter` is now four-tiered
  (`selected_actual_idx → is_current+enabled → initial_selected_idx →
  first_enabled_visible_idx`). The fall-through chain is correct as
  written but fragile: someone adding a fifth fallback that doesn't
  also gate on `enabled_actual_idx` could reintroduce focus-on-disabled.
  Worth a doc-comment on the chain explaining the invariant.
- Digit-key shortcuts only cover `1`–`9`. Lists with >9 enabled rows
  get partial keyboard-shortcut access — that's not new, but the new
  numbering hides it slightly because previously a >9-row list at least
  numbered all positions consistently with the visible row index.
- `enabled_row_number_width` uses `.max(1)` to floor at width 1. If a
  list has zero enabled rows the width is `"1".len() == 1`, which is
  fine, but the rendered output won't make it visually obvious that the
  list is non-actionable. A "no enabled options" hint row would be
  clearer.
- The renumbering changes the *visual contract* the user sees. Anyone
  who muscle-memorized "always press 3 for USB Mic" because that was
  its visible number for the past N releases will now press 3 and get
  whatever is currently at enabled-position 3, which may have shifted
  if a disabled row above it transitioned to enabled.

## Suggested follow-ups

- Add a doc-comment to `apply_filter` listing the four-tier precedence
  and the "every tier must filter through `enabled_actual_idx`"
  invariant.
- Consider extending shortcut coverage past 9 (e.g. letters, or
  Alt+digit for 10-19) for long lists like model picker.
- Add a test where every row is disabled — `select_first_enabled_row`
  falls back to `(!self.filtered_indices.is_empty()).then_some(0)`,
  which selects the first disabled row. That's probably benign because
  `accept` already gates on enabled, but worth pinning down.
- Audit other widgets in `bottom_pane/` for the same inlined disabled
  check — apply `item_is_enabled` consistently to prevent the same drift
  from re-emerging.
