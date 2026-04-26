---
pr: 19537
repo: openai/codex
sha: 8b84a97a068d349c7720f5e19dcc381d8cad08d7
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex #19537 — [mcp] Fix plugin MCP approval policy

- **Author**: mzeng-openai
- **Head SHA**: 8b84a97a068d349c7720f5e19dcc381d8cad08d7
- **Size**: +478 / -26 across 7 files. Adds `PluginMcpServerConfig` to `config/src/types.rs`, overlay logic in `core-plugins/src/loader.rs`, plugin-aware approval lookup + persistence in `core/src/mcp_tool_call.rs`, regenerated `config.schema.json`, and 240+ lines of new tests.

## Scope

Plugin-contributed MCP servers were loaded from plugin manifests, not from top-level `[mcp_servers]`. Their approval preferences had nowhere to live, so "Always allow" persistence wrote into a path that the next lookup didn't read from. Fix: introduce `plugins.<plugin>.mcp_servers.<server>` config block; overlay the policy onto manifest-provided server configs at plugin-load time; route lookup and persistence through both surfaces.

## Specific findings

- `config/src/types.rs:651-695` — new `PluginMcpServerConfig` mirrors `McpServerConfig`'s policy fields (`enabled`, `default_tools_approval_mode`, `enabled_tools`, `disabled_tools`, `tools`) but deliberately excludes transport (`type`, `url`, `command`, ...). The doc comment correctly explains why: plugin manifest owns transport, user config owns policy. Good split. `Default` impl sets `enabled: true` — consistent with `default_enabled`. `#[schemars(deny_unknown_fields)]` is a nice belt-and-braces.
- `core-plugins/src/loader.rs:533-541` — overlay loop runs per `(name, mut config)` from `plugin_mcp.mcp_servers`. The `apply_plugin_mcp_server_policy` call mutates `config` in place before it's inserted into `mcp_servers`. Order is correct. The pre-existing `if mcp_servers.insert(name.clone(), config).is_some()` warning still fires for cross-plugin name collisions but does NOT consider the user's policy intent — if user disabled the server via policy, it's still loaded then warned about. Minor.
- `core-plugins/src/loader.rs:555-573` (`apply_plugin_mcp_server_policy`) — five-step apply: `enabled`, `default_tools_approval_mode` (Some-only), `enabled_tools` (Some-only), `disabled_tools` (Some-only), per-tool `approval_mode`. The Some-only gating means policy overlay is additive: a missing field in policy preserves manifest default. **However**, `config.enabled = policy.enabled` is unconditional — if the user wrote any `[plugins.foo.mcp_servers.bar]` block (e.g. just to set `enabled_tools`), they will overwrite the manifest's `enabled` with `policy.enabled`'s default of `true`. The Default impl makes that a sensible default, but a user who wants to keep the manifest's `enabled = false` while configuring `enabled_tools` will get a surprise. Consider making `enabled` an `Option<bool>` and Some-only too, for symmetry with the rest of the overlay.
- `core/src/mcp_tool_call.rs:133-176` — call-site change: the synchronous `custom_mcp_tool_approval_mode` becomes `async` because the plugin lookup needs `plugins_manager.plugins_for_config(...).await`. All callers updated. Async-fication is a real cost — adds an `.await` on every MCP tool dispatch — but `plugins_for_config` should be cheap when the cache is warm. Verify with a microbench if dispatch latency is on the critical path.
- `core/src/mcp_tool_call.rs:691-720` — lookup precedence: user-configured `[mcp_servers.<server>]` wins; if absent, fall through to plugin overlay. Correct: explicit user-top-level config beats plugin policy beats plugin default. The `find_map(|plugin| ...)` on the plugin list returns the FIRST plugin that owns the server name. If two enabled plugins ship a server with the same name, you've already had a load-time warning, but the approval lookup will just pick whichever plugin appears first in `plugins()` ordering. Document the tie-break or assert it's deterministic.
- `core/src/mcp_tool_call.rs:1716-1755` (`persist_non_app_mcp_tool_approval`) — first checks the global/project config-folder path; if the server isn't there, falls back to writing into `plugins.<plugin>.mcp_servers.<server>.tools.<tool>.approval_mode = "approve"` in the user's `codex_home` config. Correct. The bail at `:1771` ("not configured in config.toml or an enabled plugin") is reachable if a server is loaded transiently then disabled before persistence — minor edge.
- `core/src/mcp_tool_call.rs:1696` — old `persist_custom_mcp_tool_approval` is now `#[cfg(test)]`. Confirm no other production caller exists; if not, prefer deleting it outright instead of cfg-gating.
- `core/src/mcp_tool_call_tests.rs:69-95` — new helper `write_sample_plugin_mcp` writes a plugin manifest + `.mcp.json` under `codex_home/plugins/cache/test/sample/local`. Path `plugins/cache/test/sample/local` is hardcoded — verify it matches the loader's expected layout (looks like `<source>/<name>/<channel>` or similar). If the loader's directory schema changes, this test silently breaks.
- `core/src/mcp_tool_call_tests.rs:1334-1346` — three updated assertions for `custom_mcp_tool_approval_mode` now pass `&session` and `.await`. Good.
- `core/src/mcp_tool_call_tests.rs:1364+` (test `custom_mcp_tool_approval_mode_uses_plugin_mcp_policy`) — exercises the plugin-policy path end-to-end: writes plugin files, writes `[plugins."sample@test"]` in config.toml, asserts the per-tool approval mode is read from plugin policy. Good coverage of the happy path. Missing: a test for the precedence rule (user `[mcp_servers]` wins over plugin policy when both are set), and a test for the persist→reload round-trip writing into the plugin section.
- `core/config.schema.json` — regenerated. Includes `mcp_servers` under `PluginConfig` and full `PluginMcpServerConfig` definition. `additionalProperties: false` on `PluginMcpServerConfig` (matches the `deny_unknown_fields` schemars annotation). Consistent.

## Risk

Medium-low. The fix targets a real correctness bug (silent persistence-vs-lookup mismatch) and the design is principled. Main residual risks:
1. The unconditional `config.enabled = policy.enabled` means partial policy overrides can flip `enabled` from `false` (manifest) to `true` (policy default). Surprising for users who wrote a partial overlay.
2. Async-ification of `custom_mcp_tool_approval_mode` adds an `.await` on a hot path.
3. Cross-plugin name-collision tie-break is FIFO over `plugins()` iteration order — deterministic but undocumented.

## Verdict

**merge-after-nits** — make `enabled` `Option<bool>` and only apply when `Some`, document (or assert deterministic order on) the cross-plugin collision tie-break, add a precedence-test (user top-level wins over plugin policy) and a persist→reload test, and either delete or keep-with-a-comment the now-`#[cfg(test)]`-only `persist_custom_mcp_tool_approval`. Bug-fix value is real and the test coverage is good.
