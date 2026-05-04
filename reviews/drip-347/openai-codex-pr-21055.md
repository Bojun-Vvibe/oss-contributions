# openai/codex#21055 — Preserve session MCP config on refresh

- **Head SHA**: `c511cb6b9bbad54716bf45dded81e57f2a9fc4a3`
- **Verdict**: `merge-after-nits`

## Summary

Reworks the MCP-refresh path so that per-thread session overlays (e.g., app-injected MCP servers) survive a refresh. Touches `app-server/src/codex_message_processor.rs`, the new `plugins.rs` split-out, `core/src/config/mod.rs` and `thread_manager.rs` (+351/-66 across 6 files).

## Findings

- `codex-rs/app-server/src/codex_message_processor.rs:728-757`: `effective_plugins_changed_callback` now closes over `self.config_manager.clone()` instead of a snapshot `Config`. Good — this is the bug. The previous closure captured a stale `Config` at the time of `start_remote_installed_plugins_cache_refresh`, so any subsequent disk edits were re-applied with the wrong baseline. The new path resolves config at refresh time via `ConfigManager`.
- `:5320-5328`: `mcp_server_refresh` now calls `load_latest_config(...).await?` purely for its side-effect (cache priming) and discards the result. Worth a one-line comment noting the call is intentional, otherwise a future cleanup will delete it.
- `core/src/thread_manager.rs` (per the file list): rebuild path needs to be idempotent if two refreshes race. From the diff hunk visible, the per-thread rebuild uses the thread's current `cwd`, which is correct, but please confirm there's a test exercising "refresh fires while a turn is in flight" — the PR description implies this works but the `config_tests.rs` additions look like pure-config coverage.
- The removal of `McpServerRefreshConfig` import at `:355` and dropping `queue_mcp_server_refresh_for_config` simplifies the surface area nicely. No public-API impact since both were `pub(crate)`.

## Recommendation

Land after: (1) a comment on the discarded `load_latest_config` return, (2) confirmation/test that an in-flight turn doesn't see a torn MCP server set mid-refresh. The architectural direction (config-manager-as-source-of-truth + per-thread overlay) is the right fix.
