# openai/codex#21062 — Preserve legacy MCP elicitations for Xcode 26.4

- **URL**: https://github.com/openai/codex/pull/21062
- **Head SHA**: `b37257440ee6`
- **Diffstat**: +286 / -31
- **Verdict**: `merge-after-nits`

## Summary

Plumbs an `app_server_client_name` / `app_server_client_version` pair from the app-server thread config down through `McpConnectionManager`, then routes a per-client `McpElicitationClientCompatibility` enum (currently just `Default` and `Xcode26Dot4`) into the elicitation manager so Xcode 26.4 callers continue to receive the legacy elicitation shape while everyone else gets the modern one.

## Findings

- `codex-rs/codex-mcp/src/connection_manager.rs:155-165` — `McpElicitationClientCompatibility::from_client_info` is now resolved once and threaded into both `ElicitationRequestManager::new_with_compatibility` and the per-server spawn path. Good: single source of truth, no string-sniffing inside per-server tasks.
- `codex-rs/codex-mcp/src/connection_manager_tests.rs:206-235` — `xcode_26_4_uses_legacy_elicitation_compatibility` covers `26.4`, `26.4.1`, `26.4-beta`, plus the negative cases (`26.3`, `26.5`, `27.0`, non-Xcode names, missing version). That's the right matrix; if upstream wants to be paranoid, also assert that case-folding (`xcode` lowercase) does NOT match — the matcher should stay strict.
- `codex-rs/app-server/src/request_processors/thread_processor.rs:980-984` — the now-removed `set_app_server_client_info` call site is replaced by passing the values into the new `ConfigureSession` shape. Worth a one-line note in the PR body that this changes the ordering: client info is now applied at thread construction rather than after, which is what the elicitation gate needs.
- `codex-rs/app-server/src/request_processors/external_agent_config_processor.rs:312-313` — the new fields are wired as `None` for external agents. Reasonable default, but please confirm that path can never originate from Xcode itself; otherwise external-agent flows would silently fall back to the modern shape.

## Recommendation

Right shape, right test coverage. Add the strict-name assertion and confirm the external-agent default is intentional, then merge.
