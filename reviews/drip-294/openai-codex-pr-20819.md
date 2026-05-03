# openai/codex PR #20819 — feat(tui): add raw scrollback mode

- **Repo:** openai/codex
- **PR:** #20819
- **Head SHA:** `c1c8a228564baac10d3c55dd2c7788473370dbb6`
- **Author:** fcoury-oai
- **Title:** feat(tui): add raw scrollback mode
- **Diff size:** +1138 / -56 across 38 files
- **Drip:** drip-294

## Files changed (highlights)

- `codex-rs/config/src/types.rs` (+5/-0) — adds `Tui::raw_output_mode: bool` (defaults to false).
- `codex-rs/config/src/tui_keymap.rs` (+2/-0) — adds `toggle_raw_output: Option<KeybindingsSpec>` to `TuiGlobalKeymap`.
- `codex-rs/core/src/config/mod.rs` (+8/-0) — surfaces `tui_raw_output_mode` on the runtime `Config`.
- `codex-rs/core/src/config/config_tests.rs` (+53/-0) — three new tests: defaults to false, parses true, runtime config picks it up.
- `codex-rs/core/config.schema.json` (+15/-0) — JSON-schema entries for `raw_output_mode` and the `toggle_raw_output` keybinding.
- `codex-rs/tui/src/history_cell.rs` (+538/-1) — bulk of the rendering work, plus the snapshot at `snapshots/codex_tui__history_cell__tests__raw_mode_toggle_transcript.snap` (+58/-0).
- `codex-rs/tui/src/insert_history.rs` (+58/-5), `codex-rs/tui/src/keymap.rs` (+34/-0), `codex-rs/tui/src/tui.rs` (+35/-8), `codex-rs/tui/src/streaming/controller.rs` (+53/-21) — wiring through input handling, keymap action, and stream rendering.
- `codex-rs/tui/src/chatwidget.rs` (+52/-0), `chatwidget/tests/slash_commands.rs` (+51/-0), `chatwidget/tests/status_and_layout.rs` (+41/-0) — slash-command + layout tests.

## Specific observations

- This is a sizable feature (1138 LOC across 38 files). The shape — config flag → keymap action → render-time branch in `history_cell` / `insert_history` / `streaming/controller` — is the conventional way to add a TUI display mode in this codebase, so the diff distribution looks right.
- `config/src/types.rs:617-622` — the doc comment "Start the TUI in raw scrollback mode for copy-friendly transcript output. Defaults to `false`." is clear. Good that the default is `false` so existing users see no behavior change.
- `core/src/config/config_tests.rs:663-708` — the three new tests cover (a) absence-of-key → false, (b) explicit-true parses, and (c) runtime config plumbs the flag through `Config::load_from_base_config_with_overrides`. That's the right test triad for a new config knob.
- The keymap action `toggle_raw_output` is added to `TuiGlobalKeymap` (not Chat-context), meaning it's available regardless of which pane has focus. Worth confirming in the PR description that this is intentional (vs scoping to the transcript pane only).
- `core/config.schema.json:2941-2949` — schema entry uses `allOf: [$ref: KeybindingsSpec]` which matches the surrounding pattern. Good.
- The keymap snapshot updates (`keymap_picker_*.snap`) are mechanical — adding one more action shifts the picker layout. Reviewers should sanity-check that none of the snapshot diffs lost an unrelated entry; the listed `+1`/`-1` line counts suggest only the new action was inserted, but a maintainer should still skim each snap.
- No tests for the actual raw-rendering output beyond the one new snapshot (`raw_mode_toggle_transcript.snap`). Given this is a 538-line addition to `history_cell.rs`, two or three more snap tests covering edge cases (empty history, mid-stream toggle, image cells in raw mode) would lock in the rendering contract.
- `streaming/controller.rs` net +32 lines suggests the streaming path now branches on raw-mode. This is the highest-risk file in the PR — any regression here affects every user, not just raw-mode users. Worth a careful read by a streaming-controller owner.

## Verdict: `merge-after-nits`

Solid feature, conventional shape, good config-test coverage. Three asks before merging: (1) more snap coverage of the raw-render path in `history_cell`, (2) explicit owner review of `streaming/controller.rs` deltas, (3) PR description should clarify why the keymap is global vs scoped. None of these block on their own.
