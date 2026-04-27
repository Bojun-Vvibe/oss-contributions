---
pr: https://github.com/openai/codex/pull/19778
sha: d5539f43
diff: +1811/-75
state: OPEN
---

## Summary

Third installment of the plugin-discovered-hooks surface (after #19705 plugin discovery and #19712 protocol scaffolding) — adds the actual `hooks/list` and `hooks/config/write` JSON-RPC handlers in `codex_message_processor.rs`, the user-config write path through `core/src/config/edit.rs` and `core/src/config/hook_config.rs`, and crucially an end-to-end integration test that pins the list → disable → list-shows-disabled round-trip the previous PRs were missing.

## Specific observations

- `codex-rs/app-server/tests/suite/v2/hooks_list.rs:1517-1577` — the new `hooks_list_shows_discovered_hook_and_config_write_disables_it` test is the right shape: writes a `[[hooks.PreToolUse]]` block to user `config.toml` via `write_user_hook_config()`, sends `hooks/list` and asserts the discovered hook surfaces with `enabled: true`, sends `hooks/config/write { key: hook.key, enabled: false }`, asserts `effective_enabled: false` in the response, then sends `hooks/list` again and asserts the same key now reports `enabled: false`. This directly closes the round-trip-missing critique that drip-101 raised against #19705 and drip-102 raised against #19712.
- `codex-rs/protocol/src/v2.rs` `HooksListEntry` (per `HooksListEntry.ts:7`) carries an explicit `enabled: bool` field on each entry — also closes drip-102's "schema doesn't appear to confirm per-entry enabled flag" critique. The field is a flat bool, not an enum, so a future tri-state ("enabled-by-default vs explicitly-enabled vs explicitly-disabled") would be a breaking schema change; given the config model only supports binary disable, this is acceptable for v1.
- `codex-rs/core/src/config/hook_config.rs:1715-1820` adds `set_hook_config(&mut self, key: String, enabled: bool) -> bool` plus `set_hook_config_writes_disabled_entry` and `set_hook_config_removes_entry_when_enabled` unit tests. The remove-on-enable behavior is the right call (default state is enabled, so an `enabled: true` entry is dead config) but the *first time* a user enables a hook there is no `[[hooks.config]]` entry to remove, and the `bool` return distinguishes wrote-vs-noop — worth a one-line doc comment on the function spelling out the three cases.
- The "stable hook key" format that drip-101 and drip-102 both flagged as undefined still isn't documented in either the `HookListEntry` schema or the `[[hooks.config]]` TOML round-trip helpers. The integration test at `:1574` cargo-cults `hook.key.clone()` from the list response into the write request — which is correct usage — but a downstream tool author writing a third-party hook manager has no spec for what survives across plugin reloads, codex upgrades, or hook-block reorderings within the same source file.
- `codex-rs/app-server/src/codex_message_processor.rs:1256-1391` adds both handlers; the `hooks_list` impl defaults `cwds` to the session cwd when empty (matching `HooksListParams.json:14`'s description) and `hooks_config_write` returns `effective_enabled` so the caller can re-render UI state without round-tripping through another `hooks/list`. No GC story for orphaned `[[hooks.config]]` entries when a plugin uninstalls — same gap as drip-101 noted on #19705.

## Verdict

`merge-after-nits` — solid feature-completion PR that closes the two biggest documentation gaps from the earlier stack (round-trip integration test + per-entry `enabled` schema field) and lands the user-write surface with sound TOML handling. The "stable hook key" format and plugin-uninstall GC story are still both undefined three PRs into this stack and should be pinned in the public README under `app-server/README.md` (which this PR touches at +55 lines but doesn't cover those topics) before the surface is announced.
