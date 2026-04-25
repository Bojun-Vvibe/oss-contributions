# openai/codex PR #19537 — Add plugin MCP policy persistence

- **URL:** https://github.com/openai/codex/pull/19537
- **Head SHA:** `fd8c33a5d66888324a0656ede3a440c3274c430d`
- **Files touched:** 6 (config types, plugin loader, schema, mcp_tool_call + tests)
- **Verdict:** `merge-after-nits`

## Summary

Adds a `plugins.<plugin>.mcp_servers.<name>` overlay so plugin-provided MCP
servers honour user-configured enablement and per-tool approval (including
"Always allow"), persisting the writes back into the owning plugin config
instead of dropping them on the floor.

## Specific references

- `codex-rs/config/src/types.rs:643-690` — new `PluginMcpServerConfig` with
  `enabled`, `default_tools_approval_mode`, `enabled_tools`, `disabled_tools`,
  and per-tool `tools` map. `Default::default()` correctly mirrors the
  `default_enabled` helper (`enabled: true`).
- `codex-rs/core-plugins/src/loader.rs:534-572` — `apply_plugin_mcp_server_policy`
  is called during `load_plugin`, after parsing the plugin's own
  `mcp_servers.json`. Note: it unconditionally overwrites
  `config.enabled = policy.enabled`, which means a manifest-default of `true`
  will always be replaced by the policy default of `true` — fine, but worth a
  comment because it implies `policy.enabled = false` is the only meaningful
  override.
- `codex-rs/core/src/mcp_tool_call.rs:670-712` — `custom_mcp_tool_approval_mode`
  is now `async` and takes `&Session`, so persistence can write back through
  the session. Good shape.
- `codex-rs/core/src/mcp_tool_call_tests.rs:+118` and
  `codex-rs/core/src/plugins/manager_tests.rs:+70` — tests cover policy
  loading, approval lookup, and persistence write-back. Solid coverage.

## Reasoning / nits

- `apply_plugin_mcp_server_policy` silently overwrites
  `default_tools_approval_mode` only when `Some` but always overwrites
  `enabled`. Recommend matching the semantics: only overwrite when the user
  explicitly set it (e.g. wrap `enabled` in `Option<bool>` or document the
  current behaviour).
- Schema generation in `config.schema.json` (+48 lines) looks consistent;
  worth a `just write-config-schema` re-run in CI to prevent drift.
- No migration story for users who already had ad-hoc per-server overrides
  in plugin manifests — call this out in the changelog.

Approve after the `enabled`-overwrite nit is addressed (or documented).
