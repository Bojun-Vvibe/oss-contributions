# PR #19840 — Add hook config write API

- **Repo**: openai/codex
- **PR**: #19840
- **Head SHA**: `b1307c25`
- **Author**: abhinav-oai
- **Size**: +849 / -24 across 29 files
- **URL**: https://github.com/openai/codex/pull/19840
- **Verdict**: **merge-after-nits**

## Summary

Adds a new `hooks/config/write` v2 app-server endpoint that lets
clients enable/disable individual discovered hooks by a stable `key`,
mirroring the existing `skills/config/write` shape. Storage is a
`[[hooks.config]]` array-of-tables in `~/.codex/config.toml` where
each disabled hook is persisted as `{ key = "...", enabled = false }`.
Re-enabling removes the entry (so the hook's discovery-time default
applies again), matching the skills config behavior. Filter is applied
at discovery time so disabled hooks are still *returned* by `hooks/list`
with `enabled: false` (so the UI can render them), but the
`ConfiguredHandler` execution list omits them entirely.

## Specific changes

- **Protocol**: new `HooksConfigWriteParams { key: String, enabled: bool }`
  / `HooksConfigWriteResponse { effective_enabled: bool }` types in
  `app-server-protocol/src/protocol/v2.rs:445-455`, registered in
  `protocol/common.rs:447` via `client_request_definitions!{ ... HooksConfigWrite => "hooks/config/write" { params: v2::HooksConfigWriteParams, response: v2::HooksConfigWriteResponse } }`.
  Generated JSON + TS schemas across 8 mechanical files.
- **`HookListEntry` gains two fields**: `key: String` and `enabled: bool`,
  surfaced through `HookMetadata` at
  `app-server/src/codex_message_processor.rs:9516,9528`.
- **Handler dispatch** at `codex_message_processor.rs:1082-1085` matches
  `ClientRequest::HooksConfigWrite { request_id, params }` and forwards
  to `hooks_config_write(...)` at `:7181-7222`. The handler:
  trims-and-rejects empty keys with `INVALID_PARAMS_ERROR_CODE`,
  applies a `ConfigEditsBuilder::new(&codex_home).with_edits(vec![ConfigEdit::SetHookConfig { key, enabled }]).apply()`,
  on success calls `clear_plugin_related_caches()` and returns
  `HooksConfigWriteResponse { effective_enabled: enabled }`, on failure
  wraps as `INTERNAL_ERROR_CODE`.
- **Config edit primitive** at `core/src/config/edit.rs:64,524-526,727-841`:
  new `ConfigEdit::SetHookConfig { key, enabled }` variant + a
  `set_hook_config(...)` impl that walks/creates the `hooks` table and
  `hooks.config` array-of-tables, finds an existing entry by `key`, and
  mutates: enabled → remove the negative override (and prune
  `hooks.config` / `hooks` if they become empty), disabled → upsert
  `{ key, enabled = false }`. The implementation has the same
  defer-parent-table-removal pattern as `set_skill_config` (deferred
  with `remove_hooks_table = true` flag because the nested `&mut`
  borrows on the array overlap with the parent-table removal).
- **Discovery integration** at `hooks/src/engine/discovery.rs:35-235`:
  threads a new `key_prefix: String` through `HookHandlerSource` so each
  matcher group's handler gets a stable composite key
  `format!("{key_prefix}:{event_label}:{group_index}:{handler_index}")`
  (`:447-453`). For per-source files the prefix is
  `format!("path:{}", source_path.display())`, for plugin sources it's
  `format!("plugin:{plugin_id}:{source_relative_path}")`. Then the
  `enabled` evaluation at `:454`:
  `let enabled = source.is_managed || source.hook_config_rules.is_enabled(&key);`
  — managed (project-required) hooks are always-on regardless of
  user override (correct), and the `enabled` field is added to the
  emitted `HookListEntry` (`:465`). Critically, the `ConfiguredHandler`
  push at `:476` is now wrapped in `if enabled { ... }` so disabled
  hooks don't reach execution, and `display_order` still increments
  in both branches so re-enabling preserves order.
- **Config-rules layer** at `core/src/config/.../config_rules.rs` (file
  not in diff snippet but referenced) walks the layer stack and folds
  enabled/disabled overrides per-key with a "later layers win" rule
  (an explicit `enabled = true` later removes a disabled entry from
  earlier layers — diff at `cx-19840:1109-1124`).

## Risks

- **The hook key is positional, not durable** — the diff itself
  flags this with `// TODO(abhinav): replace this positional selector
  with a durable hook id.` at the format! site
  (`hooks/src/engine/discovery.rs:441`). Today the key is
  `path:.../config.toml:pre_tool_use:0:0` — i.e. *index 0 of matcher
  group 0 of pre_tool_use in this file*. Reorder the hooks in
  `config.toml`, or insert a new hook before an existing one, and
  every persisted disabled-key in `~/.codex/config.toml` now points
  at a *different* hook than the user intended. This is a real
  surprise — the user thinks they disabled hook A but reordering
  silently re-enables A and disables B. Should at minimum be called
  out in the PR body / release notes ("disabled hooks are tracked
  positionally; reorder at your own risk") and ideally the TODO
  should be addressed before this ships, e.g. by hashing
  `(matcher, command, timeout_sec)` into the key tail so the key
  survives reordering.
- **Plugin key includes `source_relative_path`** which is more stable
  than positional — but still positional within the plugin file
  (`:0:0`), so plugin upgrades that reorder the file silently re-shuffle
  user disable state too.
- **`clear_plugin_related_caches()` after every write** — correct for
  invalidation but the cache reset isn't scoped to "the file this hook
  came from". On a workspace with many plugins, every hook
  enable/disable busts the entire plugin cache. Acceptable for the v1
  shape (UI clicks are rare) but worth a `// TODO: scope cache invalidation`
  comment.
- **No round-trip integration test** in the visible diff slice that
  asserts: write `{ key: K, enabled: false }` → reload discovery →
  `hooks/list` shows `enabled: false` for K → the corresponding
  `ConfiguredHandler` is not in the execution list. The unit tests
  pin the TOML edit shape (`set_hook_config_writes_disabled_entry`,
  `set_hook_config_removes_entry_when_enabled` at edit_tests.rs) and
  the discovery-side filter (`discovered.hook_entries[1].enabled == false`
  at `:1569-1570`), but the wiring between the two — that the
  *same* key the user writes is the *same* key discovery surfaces — is
  the most fragile part because of the positional-key issue above
  and is not pinned by an end-to-end test.

## Verdict

**merge-after-nits**: the API surface is the right shape (mirrors
`skills/config/write`), the negative-overrides-only encoding matches
the existing skills convention, the managed-hooks-bypass-user-config
rule at `discovery.rs:454` is correct security policy, and the
defer-parent-removal dance in `set_hook_config` correctly avoids
the borrow-checker trap. Two real nits before merge: (1) the
positional-key TODO needs at minimum a release-note callout, ideally
a content-derived suffix to make keys reorder-stable; (2) add an
end-to-end test that round-trips one hook disable through write →
discovery reload → `hooks/list` to pin the key contract. The cache
invalidation scope is a follow-up, not a blocker.

## What I learned

The "negative overrides only" encoding for enable/disable state —
where `enabled = false` is persisted as a row in
`[[hooks.config]]` and `enabled = true` is encoded as *absence* of a
row — is the right shape because it (a) makes the user's config
diff-readable as "disabled hooks: A, B, C" rather than "explicit
state for every hook in the system", (b) lets new discovered hooks
default-enabled without requiring the user to touch their config,
(c) keeps the config small even when most hooks are enabled. The
cost is a slightly subtler invariant — re-enabling means
*removing* a row and pruning the empty parent table, which is what
the `remove_hooks_table` deferred flag handles. The positional-key
problem is the kind of design choice that's invisible in v1
(UI works fine for the user who hasn't reordered anything) and
becomes a support ticket avalanche in v2 when users start editing
their `config.toml` by hand. The right fix is probably
`format!("...:{event}:{matcher_hash}:{command_hash}")` so the key
is content-derived and survives reordering, with a one-time
migration on first write that rewrites positional keys to
content-derived ones.
