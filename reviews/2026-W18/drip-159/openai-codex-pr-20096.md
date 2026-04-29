# Review: openai/codex#20096 — feat: Use remote installed plugin cache for skills and MCP

- **Author**: xl-openai
- **Head SHA**: `108b13ded3f27fe31dee8e453832dc39627bbec5`
- **Diff**: +747 / −53 across 9 files
- **Draft**: no

## What it does

Restructures the plugin-system warmup and cache-invalidation pipeline so that remotely-installed plugins (the `chatgpt-global` cache) are loaded into the active Codex session's skill/MCP world without requiring a local marketplace entry, and so that any plugin lifecycle event (install / uninstall / list / startup) routes through a single async invalidation callback that clears caches and queues an MCP refresh.

Before this PR there were two cache layers (`plugins_manager().clear_cache()` + `skills_manager().clear_cache()`) and they were called *synchronously* and *manually* from each plugin lifecycle handler in `codex_message_processor/plugins.rs`, with each handler then *also* manually calling `queue_mcp_server_refresh_for_config(&self, &config)`. That left ~four call sites with subtly different sequencing, and remote-installed plugins specifically bypassed the marketplace path and so never participated in skill discovery for `skills/list` until the next restart.

The fix introduces:

1. **`effective_plugins_changed_callback(&self, config: Config) -> Arc<dyn Fn() + Send + Sync>`** at `codex_message_processor.rs:725-733` — a thread-safe callback factory that closes over `Arc::clone(&self.thread_manager)` and the config. The returned `Arc<Fn>` is what downstream plugin-manager background tasks call when the *effective* plugin set changes (i.e., something a thread would actually see).
2. **`spawn_effective_plugins_changed_task`** at `codex_message_processor.rs:739-752` — `tokio::spawn` of a future that clears both caches, returns early when there are no live threads (avoiding pointless MCP-refresh fanout), and otherwise calls `Self::queue_mcp_server_refresh_for_config(&thread_manager, &config).await` (note: the latter is now an associated function, not `&self`-bound, precisely so this callback can be called from contexts where `self` isn't available).
3. **`queue_mcp_server_refresh_for_config` is hoisted to take `&Arc<ThreadManager>`** (`codex_message_processor.rs:5401-5406`) instead of `&self`. Internal call sites get one extra `Arc::clone`, but the function is now usable from spawned tasks and from `PluginManager` background callbacks.
4. **All four lifecycle handlers in `plugins.rs` now route through `on_effective_plugins_changed(config)`** — `plugin_install` (line 367), `plugin_install_remote` (now goes through `maybe_start_remote_installed_plugins_cache_refresh_after_mutation` with the callback baked in, and removes the inline `queue_mcp_server_refresh_for_config` call), `plugin_uninstall` (line 586-595, with a config-reload-failure fallback to `clear_plugin_related_caches()` and a `warn!` log), and `plugin_uninstall_remote` (line 681-705, with a `clear_remote_installed_plugins_cache()` short-circuit that fires `on_effective_plugins_changed` only if it actually invalidated).
5. **`message_processor.rs:317-326`** wires the callback into the *startup* warmup: `plugins_manager().maybe_start_plugin_startup_tasks_for_config(&config, auth_manager.clone(), Some(on_effective_plugins_changed))`. Same pattern at `plugins.rs:42-47` for `maybe_start_plugin_list_background_tasks_for_config`.
6. **`core/src/plugins/manager.rs` (+281 / −8)** is the bulk of the diff — adds a remote-installed-plugins cache, the `maybe_start_remote_installed_plugins_cache_refresh_after_mutation` entry point, and a `clear_remote_installed_plugins_cache` accessor that returns whether a clear actually happened.
7. **`core/src/plugins/manager_tests.rs`** (+63) and **`app-server/tests/suite/v2/skills_list.rs`** (+217) — the latter adds `skills_list_loads_remote_installed_plugin_skills_from_cache`, which writes a cached remote plugin under `codex_home/plugins/cache/chatgpt-global/linear/local/skills/triage-issues/SKILL.md`, mocks the `/backend-api/ps/plugins/list` endpoint with realistic GLOBAL+WORKSPACE responses, and asserts that `skills/list` finds the `triage-issues` skill after the cache warms.
8. **`core/src/session/handlers.rs`** (+1 / −5) — small simplification, presumably because the manual clear+refresh is now handled by the callback.
9. **`core-plugins/src/loader.rs:39-42`** and **`core-plugins/src/remote.rs:57-0`** — loader changes to read remote-installed-plugins from the cache and a new remote helper to manage that cache.

The behavioral signature change at `codex_message_processor.rs:6286-6294` is also worth flagging: `effective_skill_roots_for_layer_stack` lost its `bool` argument and now takes `&config` directly; the caller hoists the `workspace_codex_plugins_enabled` gate up one level into an `if/else` that returns `Vec::new()` when plugins aren't enabled.

## Concerns

1. **`spawn_effective_plugins_changed_task` does fire-and-forget `tokio::spawn` with no error surface.** If `queue_mcp_server_refresh_for_config` fails, the only signal is `warn!("failed to queue MCP refresh after effective plugins changed: {err:?}")`. There's no test that pins this — every callback-driven test path I can see in the diff uses the success path. A test that injects a mock `ThreadManager` whose `refresh_mcp_servers` fails would lock down the warn-and-swallow contract.

2. **Race window between cache-clear and refresh-queue.** `spawn_effective_plugins_changed_task` does `clear_cache()` *then* checks `list_thread_ids().await.is_empty()` *then* queues refresh. A thread that starts a turn between the cache clear and the empty check will (a) see an empty cache (correct, will repopulate from disk) but (b) also receive an MCP refresh for the new effective set — possibly redundant, possibly conflicting if it raced its own cache repopulation. The pre-PR sequence was synchronous-relative-to-the-handler so this race didn't exist. Worth a comment explaining whether the refresh is idempotent under that race.

3. **`plugin_uninstall_remote` only fires `on_effective_plugins_changed` when `clear_remote_installed_plugins_cache()` returns true** (line 691-697). If the remote-uninstall API call succeeded *but* the local cache had nothing to clear (e.g., the plugin was already uninstalled locally), the callback doesn't fire — but a `maybe_start_remote_installed_plugins_cache_refresh_after_mutation` does. Asymmetric: callers will see "I uninstalled, why didn't a refresh happen?" Worth either firing the callback unconditionally on `Ok(())` or documenting the short-circuit.

4. **The 217-line `skills_list_loads_remote_installed_plugin_skills_from_cache` test is doing a lot.** It mocks two pages (GLOBAL + WORKSPACE), pre-seeds a real on-disk plugin layout (`plugins/cache/chatgpt-global/linear/local/.codex-plugin/plugin.json`), and uses `wiremock` to assert backend traffic. That's the right shape for an integration test, but the brittleness is in the cache path string `chatgpt-global/linear/local` — if the cache layout changes, the test silently asserts on a non-existent path and passes vacuously. A `std::fs::canonicalize(...)` wrapping that asserts the file actually exists at the expected location would tighten it.

5. **Hoisted-out `if workspace_codex_plugins_enabled` at `codex_message_processor.rs:6289`.** Pre-PR, `effective_skill_roots_for_layer_stack` took the bool inline; post-PR, the caller branches and returns `Vec::new()` when plugins are off. Behaviorally equivalent for this caller — but if any *other* caller of `effective_skill_roots_for_layer_stack` exists in the tree and relied on the old bool gate being inside the function, that caller now has to add its own gate or it'll get the full skill-roots set even when plugins are off. The diff doesn't show an audit. `rg 'effective_skill_roots_for_layer_stack' codex-rs/` would catch it.

## Verdict

**merge-after-nits** — solid consolidation around a single cache-invalidation callback that fixes the real "remote-installed plugins don't show up in `skills/list`" bug, with a credible integration test. The async fire-and-forget callback semantics need at least one comment explaining idempotency-under-race, and the asymmetric uninstall-callback behavior at `plugins.rs:691-697` should either be unified or commented. The `effective_skill_roots_for_layer_stack` signature change deserves a one-grep audit before landing.

