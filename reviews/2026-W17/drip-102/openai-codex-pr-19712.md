# openai/codex PR #19712 — Add hook enablement config APIs

- Link: https://github.com/openai/codex/pull/19712
- Head SHA: `24c0dcc4`
- Size: large (3761-line diff across schema + protocol + core + tests)

## Summary

Adds the missing config-write surface for the per-hook enable/disable toggle that the plugin-discovered-hooks feature (#19705) needed but didn't ship. Introduces `hooks/list` and `hooks/config/write` JSON-RPC methods on the app-server, with a `HookConfigSelector` enum tagged by `source` (`plugin` requires `pluginId` + `key`, `user`/`project` requires `key` only — visible at `app-server-protocol/schema/json/HooksConfigWriteParams.json:21-104` as `oneOf`-discriminated variants), and persists the toggle into the user/project `[[hooks.config]]` TOML block.

## Specific-line citations

- `ClientRequest.json:21-104` defines `HooksConfigWriteParams` as a closed `oneOf` over `{source: "plugin", enabled, key, pluginId}` and `{source: "user"|"project", enabled, key}` — `additionalProperties: false` on each variant means the schema is forward-incompatible with new selector types without a v3 bump, which is the right tradeoff for a typed config API.
- `app-server/src/v2/hooks.rs` (new module, sketched in the diff at `:1570`) adds `async fn hooks_list(&self, request_id, params: HooksListParams)` and `:1652` adds `async fn hooks_config_write(&self, request_id, params: HooksConfigWriteParams)`. The list endpoint preserves disabled-by-config entries (paralleling the `skills/list` shape — see `hook_to_info` helper at `:1743`).
- `core/src/config/hook_config.rs` (new file at the diff trailer) introduces `HookConfigSelector`, `set_hook_config(&mut self, selector, enabled) -> bool` at `:2140`, plus three TOML round-trip helpers — `hook_config_selector_from_table` (`:2280`), `write_hook_config_selector` (`:2322`), and `normalize_hook_config_selector` (`:2345`) — with the path-normalization helper `normalize_hook_config_source_path` (`:2362`) that is critical for the project-source variant (otherwise `./project` and an absolute path to the same dir would create duplicate config entries).
- `core/src/config/edit.rs` is the persistence path that writes `[[hooks.config]]` blocks into the user-level config TOML.
- `core/src/config/hook_events.rs:85-117` adds `hook_events_deserialize_config_overrides` and `:91-117` `hook_events_deserialize_project_config_overrides` tests — pin the deserializer for both user and project source variants.
- `app-server/tests/suite/v2/hooks_config_write.rs:1842` adds `hooks_config_write_persists_project_selector` end-to-end test, and `hooks_list_returns_user_config_hooks` at `:1795` covers the read side.
- `app-server-protocol/src/protocol.rs:1140-1141` adds the two new params types to the `ClientRequest` enum, wired as `"hooks/list"` and `"hooks/config/write"` methods.
- Round-trip test for the discriminated-union shape: `hooks_config_write_params_round_trips_selector_variants` at `:1397` — pins that JSON `{source:"plugin", pluginId, key, enabled}` ↔ Rust `HookConfigSelector::Plugin{...}` is reversible, which is the right level of test for a discriminated union exposed over JSON-RPC.

## Verdict

**merge-after-nits**

## Rationale

The protocol design is correct: the `oneOf`-by-`source` selector is the right shape for "hooks live in three places (user, project, plugin) but a write must address exactly one of them", `additionalProperties: false` keeps the contract tight, and the `set_hook_config_writes_disabled_plugin_entry` test (`:2386`) pins the specific use case that #19705 wired the UI for. Splitting `hook_config.rs` (data + TOML round-trip) from `edit.rs` (persistence) keeps the per-file responsibilities narrow.

Nits:

1. **No documented contract on selector key stability.** The `key` field carries the "stable hook key" referenced in #19705's PR body, but neither this PR nor #19705 defines what makes a key stable across plugin updates. If a plugin renames a hook, does the old `[[hooks.config]]` entry orphan silently or migrate? The persistence test at `:1842` only covers the happy path. Worth a one-paragraph note in the schema doc on key-stability guarantees.
2. **No GC for orphaned entries.** `set_hook_config` writes to `[[hooks.config]]` but nothing in this PR collects entries whose `pluginId`/`key` no longer resolves (e.g. plugin uninstalled). Eventually `[[hooks.config]]` becomes a graveyard. This is consistent with how `[[skills.config]]` works today, so it's not a blocker — but it's a known limitation worth tracking as a follow-up.
3. **`normalize_hook_config_source_path`** strips trailing separators and canonicalizes — but it's not clear from the diff snippet whether it follows symlinks (`fs::canonicalize`) or just lexically normalizes. If it follows symlinks, a project moved across symlinks loses its `[[hooks.config]]` association. If it just lexically normalizes, two paths that resolve to the same directory but differ in symlink chain create two entries. Worth a comment naming which behavior was chosen and why.
4. The `hooks/list` response shape preserves disabled-by-user-config entries but the schema for `HooksListResponse` (referenced via `HookSource` at `:297`/`:619`/`:1077`/`:1174`) doesn't appear in the diff trailer to confirm it carries an explicit `enabled: bool` per entry — needed so clients can render the disabled state without a second lookup.

None of these are blockers; the protocol surface and persistence are sound, and the round-trip test pins the wire-level contract.
