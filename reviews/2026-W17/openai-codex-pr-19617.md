# openai/codex #19617 — feat(tui): add codex logo to session header

- **Repo**: openai/codex
- **PR**: #19617
- **Author**: fcoury-oai (Felipe Coury)
- **Head SHA**: a166d6d03f1781a297c182e3439758f6916c7c1f
- **Base**: main
- **Size**: +187 / −30 in `codex-rs/tui/src/history_cell.rs` (+143/−5) plus
  4 snapshot updates.

## What it changes

Adds an 8-line ASCII-art codex logo to the left of the session header,
with two color gradients (`CODEX_LOGO_BRIGHT_GRADIENT` for dark
terminals, `CODEX_LOGO_DARK_GRADIENT` for light) chosen via
`codex_logo_gradient_for_bg`. Header layout is split into:

```rust
const CODEX_LOGO_WIDTH: usize = 16;
const CODEX_LOGO_GAP_WIDTH: usize = 2;
const SESSION_HEADER_TEXT_MAX_INNER_WIDTH: usize = 56;
pub(crate) const SESSION_HEADER_MAX_INNER_WIDTH: usize =
    CODEX_LOGO_WIDTH + CODEX_LOGO_GAP_WIDTH + SESSION_HEADER_TEXT_MAX_INNER_WIDTH;
```

The header gracefully degrades to text-only when terminal width is
below `SESSION_HEADER_MAX_INNER_WIDTH + 4` (line ~1438).

## Strengths

- Width-responsive layout: `let show_logo = usize::from(width) >= …`
  ensures narrow terminals don't get a clipped logo. Falls back to
  the previous text-only `SESSION_HEADER_TEXT_MAX_INNER_WIDTH = 56`
  layout, so nothing regresses for users on small panes.
- Light/dark adaptation via `terminal_palette` + `crate::color::is_light`
  is the right hook — uses the existing palette detection rather than
  forcing a setting.
- `padded_logo_line` correctly uses `UnicodeWidthStr::width` to handle
  the wide block characters (the logo uses U+28xx Braille glyphs which
  the terminal may render at varying widths).
- Snapshot tests are updated for all four affected scenarios
  (`clear_ui_after_long_transcript_fresh_header_only`,
  `clear_ui_header_fast_status_fast_capable_models`,
  `session_header_indicates_yolo_mode`,
  `session_info_availability_nux_tooltip_snapshot`). That's
  comprehensive snapshot hygiene.

## Concerns / asks

- The Braille logo will render *very* differently across fonts. On
  fonts without good U+2800 block coverage (some legacy Linux
  terminal fonts, older Windows console fonts) it may show as
  tofu boxes. A fallback to an ASCII-only logo when terminal font
  isn't known to support Braille would be nice but is hard to
  detect reliably — at minimum, an env var or config to disable
  the logo would help users on such terminals.
- The two gradient palettes are hard-coded blue/purple. If users
  themed their terminal expecting custom branding (e.g. corporate
  Codex deployments), they have no override. A
  `[tui.session_header.logo]` config knob (`enabled`, `gradient`)
  would be a small follow-up.
- `CODEX_LOGO_WIDTH = 16` is asserted by eyeballing; if any of the
  `CODEX_LOGO_LINES` exceeds 16 visual cells, `padded_logo_line`
  would saturate to 0 padding silently. A `debug_assert!` at startup
  validating each line's `UnicodeWidthStr::width <= CODEX_LOGO_WIDTH`
  would catch future logo edits at test time.

## Verdict

`merge-after-nits` — solid implementation with good responsive
fallback and snapshot coverage. Asks: a debug-assert on logo line
widths and a follow-up tracking issue for "disable logo" config.
Pure cosmetic feature, low risk to merge.
