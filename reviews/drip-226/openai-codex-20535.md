# openai/codex#20535 — fix(tui): add alt-enter newline alias

- Link: https://github.com/openai/codex/pull/20535
- Head SHA: `7b7764f29d7f28112ebc6cb08e67d6dfd7c96912`
- Files: `tui/src/keymap.rs` (+15/−10), `tui/src/keymap_setup.rs` (+9/−9), `tui/src/snapshots/codex_tui__keymap_setup__tests__keymap_action_menu.snap` (+1/−1)

## Notes
- One-line behavioral change at `keymap.rs:427` adds `alt(KeyCode::Enter)` to the `editor.insert_newline` default-bindings vec alongside the existing `ctrl-j`/`ctrl-m`/`enter`/`shift-enter` quartet, fixing the gap where macOS Terminal.app and most Linux terminals emit `Esc+Enter` (alt-enter) for the natural "newline in composer" gesture but the TUI was previously consuming alt-enter as a custom submit binding only — most users had no way to insert a newline mid-composer without setting a custom keymap.
- Test rewrite at `keymap.rs:1470-1481` flips the prior "default insert_newline includes shift-enter" weak assertion (which would still pass if alt-enter were silently dropped from the defaults vec) to an exact-vector equality `assert_eq!(runtime.editor.insert_newline, vec![ctrl-j, ctrl-m, enter, shift-enter, alt-enter])` — this is the right anti-behavior pin since the next reordering or addition surfaces as a test diff in code review rather than silently working in production but not matching the snapshot.
- Migration of the eight existing `keymap_setup.rs` tests at `:1099`, `:1414`, `:1535`, `:1559-1599` flips every `alt-enter` literal to either `ctrl-shift-enter` or `alt-shift-enter` so the prior tests' "configure a custom submit binding" intent survives even though their old chosen example (`alt-enter`) now collides with the new newline default — done in lockstep with the snapshot at `keymap_action_menu.snap:25` which now shows `alt-shift-enter | Replace this binding | enabled`. The lockstep migration is correct: the tests verify the keymap-config plumbing (parse, store, swap), not the semantic content of the chosen key.

## Nits
- No regression test for the load-bearing case "user has a custom keymap that *already* binds alt-enter to submit (e.g. configured before this PR landed)" — after this change their alt-enter does both submit + newline, and the resolution order is undocumented. A test arm `user_alt_enter_submit_overrides_default_newline` asserting the precedence (and the README/help-text describing it) would prevent the inevitable issue.
- The five-binding `editor.insert_newline` default is now busy enough that the `assert_eq!` at `:1470-1481` reads as fragile — a one-line `// keep in sync with default_bindings![] at L424-428` doc-comment above the test would help the next contributor adding `option-enter` not forget to update both sites.
- `alt(KeyCode::Enter)` on Windows ConPTY/legacy console may not deliver as expected (terminal sends `0x0a` directly without the alt-modifier escape). Worth a one-line note in the keymap docs or a `// macOS/Linux: alt+Enter; Windows: ctrl+Enter recommended` comment near `:427` so users on Windows who file "alt-enter doesn't work" issues get redirected.
- The snapshot file is touched only at the single `alt-enter | … | enabled` row (correct), but the snapshot's stability against future re-orderings of the binding vec depends on `BTreeMap`-style iteration somewhere upstream — worth confirming the source of `replace picker:` ordering is not insertion-order-dependent.

**Verdict: merge-after-nits**
