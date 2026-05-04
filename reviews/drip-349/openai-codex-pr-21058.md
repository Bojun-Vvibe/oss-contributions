# openai/codex#21058 — fix(tui): support modified backspace/delete keys

- **URL**: https://github.com/openai/codex/pull/21058
- **Head SHA**: `1d9de78a1010`
- **Diffstat**: +111 / -1
- **Verdict**: `merge-as-is`

## Summary

Extends the default editor keymap so that Shift+Backspace / Shift+Delete behave as plain Backspace/Delete, and Ctrl+Backspace / Ctrl+Delete (plus their Ctrl+Shift variants) do word-wise deletion. Adds matching textarea unit tests and a keymap-defaults assertion.

## Findings

- `codex-rs/tui/src/keymap.rs:578-606` — Shift variants are appended to existing binding lists rather than replacing them, so users with custom overrides keep their behavior. The Ctrl+Shift entries are constructed via `raw(KeyBinding::new(..., CONTROL | SHIFT))` because the convenience helpers don't accept compound modifiers; that's the right escape hatch.
- `codex-rs/tui/src/bottom_pane/textarea.rs:2722-2768` — three new tests cover Shift+Backspace, Shift+Delete, and the Ctrl / Ctrl+Shift loop for both word-delete directions. The loop iteration over `[CONTROL, CONTROL | SHIFT]` is the cleanest way to assert the alias set without copy-paste.
- `codex-rs/tui/src/keymap.rs:1839-1885` — the new `default_editor_deletion_includes_modified_backspace_delete_aliases` test pins the defaults against future regressions; if anyone trims the binding list they'll get a clear failure.
- No conflict with existing `alt(Backspace)` / `ctrl('w')` / `ctrl('h')` bindings — they remain in place.

## Recommendation

Small, well-tested terminal-ergonomics fix. Ship.
