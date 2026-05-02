---
repo: openai/codex
pr: 20819
head_sha: c1c8a228564baac10d3c55dd2c7788473370dbb6
title: "feat(tui): add raw scrollback mode"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: openai/codex#20819 — `feat(tui): add raw scrollback mode`

**Head SHA:** `c1c8a228564baac10d3c55dd2c7788473370dbb6`
**Stat:** +1138 / −56. Author: fcoury-oai. Targets #12200, #9252, #8258
(full clean copy); partially #2880, #19820, #18979.

## What it changes

Adds a "raw scrollback" mode to the TUI so users can copy transcript
content with terminal-native selection without the rich-mode wrapping,
gutter, and indentation that breaks paragraph/command paste.

Surface area:

- **Config**: new `Tui.raw_output_mode: bool` (default `false`) at
  `codex-rs/config/src/types.rs:617–621`. Mirrored as
  `Config.tui_raw_output_mode` at `codex-rs/core/src/config/mod.rs:516–519`,
  populated from the `[tui]` table at `mod.rs:3092` (the `unwrap_or(false)`
  follows the same pattern as the surrounding `tui_vim_mode_default`
  field).
- **Schema**: regenerated `codex-rs/core/config.schema.json` adds
  `raw_output_mode` and `toggle_raw_output` keymap entries
  (lines 2455, 2554–2559, 2941–2949, 3053).
- **Keymap**: new optional `toggle_raw_output: Option<KeybindingsSpec>`
  on `TuiGlobalKeymap` at `tui_keymap.rs:104–108`. Default binding
  `alt-r` per the PR description.
- **Slash command**: `/raw [on|off]` exposes the same toggle and emits
  a confirmation message; the keybinding path toggles silently (per PR
  description). Tests
  `raw_slash_command_toggles_and_accepts_on_off_args` and
  `raw_output_toggle` exercise both paths.
- **Render path**: rich/raw-aware transcript cell rendering preserves
  source text in raw mode and lets the terminal soft-wrap.
- **Test coverage** in `config_tests.rs:550, 661, 2173, 6499, 6702,
  6859, 7001` — every existing precedence/profile test gets a
  `raw_output_mode: false` line so the deserialization match stays
  exhaustive. Plus three new dedicated tests
  (`test_tui_raw_output_mode_defaults_to_false`,
  `test_tui_raw_output_mode_true`,
  `runtime_config_uses_tui_raw_output_mode`).

Validation list in PR description includes `cargo test -p codex-tui
raw_output_toggle`, `argument-comment-lint`, snapshot check, and
`just write-config-schema` — schema regenerated in the diff confirms
the last one.

## Assessment

- The config plumbing is correct and exhaustive. Touching every
  precedence-fixture test (lines 550, 2173, 6499, 6702, 6859, 7001) is
  the right move — those tests use struct-pattern matches, so a new
  field forces compile errors until each fixture is updated. Better
  than `..Default::default()` shortcuts.
- Toggle naming is consistent: `toggle_raw_output` action,
  `tui.keymap.global.toggle_raw_output` keymap slot,
  `tui.raw_output_mode` config flag, `/raw [on|off]` slash command.
  No surprises.
- The keybinding path being silent while `/raw` emits a confirmation is
  a sensible UX choice — keybind users see the visual mode change,
  slash-command users get textual feedback.
- The PR carefully scopes what it does *not* solve in #2880 / #19820 /
  #18979 (mouse selection, copy-as-markdown, paste-into-composer
  splitting). That's appropriate scope discipline for a 1.1k-line
  feature.
- 1138 added lines is large and the diff truncated before showing the
  render-path changes, so the rich/raw transcript-cell rendering
  divergence is the highest-risk surface I couldn't fully audit here.
  The snapshot tests + the dedicated `raw_output_mode_can_change_*`
  tests in the validation list are the right sentinels for that.

## Nits

- (Non-blocking) `raw_output_mode_can_change_without_inserting_notice`
  is a great test name; consider also asserting that the *currently
  visible* transcript cells re-render to raw form on toggle (not just
  that future cells render raw). If the implementation already covers
  this via re-render-on-resize, a one-line assertion locks it in.
- (Non-blocking) Default keybinding `alt-r` could collide with
  user-defined macros; documenting the override in the keymap doc
  comment (`tui_keymap.rs:107`) would help discoverability — the
  current doc string just says "Toggle raw scrollback mode for
  copy-friendly transcript selection." without mentioning the default.

## Verdict

**`merge-after-nits`** — well-scoped feature, defensive test
coverage, schema regenerated, clear UX delineation between slash
command and keybinding. Two non-blocking polish items; otherwise ready.
