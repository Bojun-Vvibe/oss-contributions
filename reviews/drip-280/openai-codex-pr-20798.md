# Review: openai/codex #20798 — feat(tui): improve TUI keymap coverage

- **PR**: https://github.com/openai/codex/pull/20798
- **Head SHA**: `194293fac57464569f59fcfc3772971e9464f5cd`
- **Diff size**: ~1150 lines, primary files: `config/src/tui_keymap.rs`,
  `core/config.schema.json`, `tui/src/chatwidget.rs`, `tui/src/key_hint.rs`,
  `tui/src/chatwidget/tests/slash_commands.rs`

## What it changes

Three orthogonal expansions of the configurable-keymap surface:

1. **New `toggle_fast_mode` global keybinding** (`tui_keymap.rs:104-110`,
   `config.schema.json:2421` and `:2907-2913`). Routes the existing `/fast` slash command
   logic through a new `toggle_fast_mode_from_ui()` chatwidget method
   (`chatwidget.rs:178-185` replaces inline `next_tier` ternary with a single method
   call). Test added at `slash_commands.rs:1722-1751` asserts the keybinding produces the
   same `OverrideTurnContext` + `PersistServiceTierSelection` events as the slash
   command. A second test at `:1753-1765` pins the gating: the keybinding is only
   available when `Feature::FastMode` is enabled *and* the bottom pane is idle.
2. **New `kill_whole_line` editor keybinding** (`tui_keymap.rs:171-173`,
   `config.schema.json:2403`, `:2772-2779`, `:2998`). Standard editor primitive — kill
   entire current line (typically `Ctrl-K` semantics in some configs).
3. **Raw C0 control character matching in `key_hint.rs`**
   (`key_hint.rs:30-36`, `:46-49`, `:55-58`). Old code only normalized
   shifted-letter ASCII; new code also normalizes raw C0 controls so a binding defined
   as `ctrl-j` matches a raw LF byte (which is what some terminals report). Function
   renamed `normalize_shifted_ascii_char` → `normalize_key_parts` and made
   `pub(crate)`.

## Assessment

The fast-mode toggle keybinding is well-tested. The two new tests at
`slash_commands.rs:1722` and `:1753` together cover both happy path (events match slash
command) and the gating contract (`can_toggle_fast_mode_from_keybinding` returns false
when feature disabled or when task is running). That's the right shape — the natural
silent-failure mode would be "keybinding fires but produces nothing because feature is
off"; the explicit gating test catches that.

The `normalize_key_parts` change is the subtle one. Renaming a private fn and changing
its signature semantics in the same diff is fine, but the comment update at line 388-395
is the only documentation of the new C0 behavior. A small unit test in
`key_hint.rs`'s test module — "raw `KeyCode::Char('\n')` matches a binding defined as
`ctrl-j`" — would prevent the next refactor from silently dropping this. I don't see
that test in the diff window I read.

Concerns:

- **`kill_whole_line` interaction with `kill_line_start` and `kill_line_end`**: the kill
  buffer semantics matter. If `kill_whole_line` writes to the same yank buffer as the
  other two killers, and a user binds it to a key they previously had bound to
  `kill_line_end`, the yank behavior changes. Not a code defect — a release-note item.
- **No keybinding default**: from the schema diff at `:2421` the default value is `null`,
  meaning users must opt in by configuring it. That's the right call — auto-binding a key
  could collide with existing user setups — but it should be called out so users know
  the feature exists.
- **Cross-platform raw-C0 mapping**: the C0 set is 0x00–0x1F. The PR comment at
  `key_hint.rs:392-395` calls out `ctrl-j → LF` specifically. What about `ctrl-i → TAB`,
  `ctrl-m → CR`, `ctrl-h → BS`? If `normalize_key_parts` only handles `ctrl-j`,
  bindings on `ctrl-i`/`ctrl-m`/`ctrl-h` will silently behave differently across
  terminals. If it handles the full C0 range, a parametric test asserting that helps.

## Verdict

`merge-after-nits` — fast-mode keybinding is well-covered, kill-whole-line is a
straightforward addition. Add (a) a key-hint test for raw-C0 matching (at minimum the
`ctrl-j`/LF case the PR explicitly calls out), and (b) confirm in the PR body which C0
characters `normalize_key_parts` actually normalizes (just LF, or the full set?).
