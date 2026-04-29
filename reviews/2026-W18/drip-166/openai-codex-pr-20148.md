# openai/codex #20148 — Enforce remote plugin cache cleanup on account changes

- **PR:** https://github.com/openai/codex/pull/20148
- **Head SHA:** `0274a719841320c37bf4c32ff2868ac18dc49180`
- **Size:** ~+1100 / −small (new `core-plugins/src/remote_cache.rs` 181 lines + 291-line tests file + message_processor wiring + integration tests in `tests/suite/v2/account.rs`)

## Summary
Adds enforced cleanup of the on-disk remote plugin cache (`<codex_home>/plugins/cache/{chatgpt-global,chatgpt-workspace}/...`) at three account-state transitions: (1) explicit `setAuthToken` login, (2) async chatgpt OAuth login completion, (3) logout. New `core-plugins/src/remote_cache.rs` exposes two operations — `clear_remote_plugin_cache` (logout / api-key login / no auth) and `prune_remote_plugin_cache_for_current_auth` (chatgpt login: keep only what the backend reports as installed for the now-active account; clear everything if the fetch fails). This closes the data-leak class where a previous account's cached plugin bundles stay readable after switching to a different ChatGPT account.

## Specific observations

1. **`RemotePluginCacheRetainSet::from_marketplaces` at `remote_cache.rs:~648` builds the keep-set keyed on both `plugin.name` AND `plugin.id`** (`plugin_names.insert(plugin.name); plugin_names.insert(plugin.id);`). The integration test at `tests/suite/v2/account.rs:~443` verifies this both-keys behavior — `write_remote_plugin_cache(&codex_home, "chatgpt-global", "linear")` (name-keyed) and `write_remote_plugin_cache(&codex_home, "chatgpt-global", "plugins~Plugin_linear")` (id-keyed) are both retained when the backend reports `{id: "plugins~Plugin_linear", name: "linear"}`. This double-key retention is correct for the rename-during-development case but means a stale cache dir whose name happens to collide with a different-account plugin's id would be retained — extremely unlikely in practice but not impossible.

2. **Disabled remote plugins are intentionally retained** (`if plugin.installed { ... insert ... }` branches on `installed`, not `enabled`). The doc comment is explicit: "Disabled remote plugins are retained because the backend still reports them as installed; disabled state controls availability, not local cache ownership." This is the right call — disabling a plugin shouldn't cost a re-download on next enable. Worth verifying `plugin.installed` is set on the disabled-but-installed `WORKSPACE` test case (`enabled: false`, mocked at the integration test): yes, `mock_remote_installed_plugins` is on `/installed`, so `installed: true` is implicit by being in that response.

3. **Three message_processor wiring sites:**
   - **logout** (`message_processor.rs:~1890`) — unconditional `clear_remote_plugin_cache(self.config.codex_home...)` followed by `self.clear_plugin_related_caches()`. Order matters: disk first, then in-memory. Correct.
   - **`setAuthToken` API-key path** (`~1469`) — `self.enforce_remote_plugin_cache_after_auth_change().await?`. Calls `sync_remote_plugin_cache_for_config` which checks both `Feature::Plugins` AND `Feature::RemotePlugin`; if either is off, it clears (instead of pruning). Correct fail-safe.
   - **async chatgpt login completion** (`~1820`) — `Self::enforce_remote_plugin_cache_after_async_chatgpt_login` runs **before** the `AccountUpdatedNotification` is sent. So clients observing the notification can assume cache state is consistent with the new auth identity at the time they react.

4. **`fetch_remote_marketplaces` failure path falls back to `clear_remote_plugin_cache`** — `prune_remote_plugin_cache_for_current_auth` body, "If the account cannot be read, all remote plugin cache entries are removed so stale bundles from a previous account cannot stay visible locally." This is the right security default. The contrasting (wrong) choice would have been "keep the cache as-is on fetch failure" which would have leaked the previous account's cache forever if the backend was unreachable. Test coverage on this path is implicit — the integration test only mocks the success path.

5. **`run_cache_mutation` wraps each op in `tokio::task::spawn_blocking`** because `fs::remove_dir_all` is sync. Correct. The error message construction is good — `failed to join {context} task: {err}` distinguishes the join failure from the actual mutation failure.

6. **`ChatgptLoginCompletionContext` struct refactor** (introduced near `~1576` and `~1650`) deduplicates the previous five-arg call through `send_chatgpt_login_completion_notifications`. Worth it — both call sites had drifted slightly (wait, they were identical, but the refactor pre-empts future drift when the cleanup logic is added). Good preparatory step.

7. **`fallback_config: Arc<Config>` in `enforce_remote_plugin_cache_after_async_chatgpt_login`** is used when `config_manager.load_latest_config(None).await` fails post-login. The fallback warns and uses the startup config — which means the cleanup may run against a slightly-stale `chatgpt_base_url` if the user's config was edited between startup and login completion. Edge case, almost certainly fine, but worth a comment.

## Risks

- **Disk I/O on the auth-change hot path** — `fs::remove_dir_all` per-marketplace is wrapped in `spawn_blocking`, so it won't block the runtime, but a multi-GB plugin cache will introduce visible latency at logout. Probably acceptable; worth a `tracing::info!` with the elapsed time so this can be diagnosed in user reports.
- **The `installed` vs `enabled` semantics rely on a backend invariant** — if the backend ever ships a plugin row with `installed: false, enabled: true` (or omits `installed`), the retain-set logic silently treats it as "not in the keep-set" and the bundle gets deleted on next prune. Worth a defensive `installed.unwrap_or(true)` or equivalent if the backend schema allows nullability.
- **`spawn_blocking` isn't cancellable** — if the user logs out and then immediately switches accounts before the logout's cache clear finishes, both ops may race on the same directory. `fs::remove_dir_all` against a partially-deleted directory will fail with `ENOENT` for some entries; the retry sequence (clear → prune) should be ordered or guarded. Quick check: the doc comment doesn't address this; worth either a per-codex_home `Mutex` or a "last-writer-wins is fine, the worst outcome is one extra clear" rationale.
- **`clear_remote_plugin_cache_blocking` only clears `chatgpt-global` and `chatgpt-workspace`** (per `REMOTE_MARKETPLACE_NAMES`). Local-disk-only marketplaces (`openai-curated`, third-party) are intentionally untouched — verified by the `assert!(...).join("plugins/cache/openai-curated/gmail").is_dir())` at the integration test. Correct scope.

## Suggestions

- **(Recommended)** Add a `tracing::info!` log inside `clear_remote_plugin_cache_blocking` and `prune_remote_plugin_cache_blocking` with elapsed time and entries-removed count, so support can diagnose "logout takes 30s" reports. Currently the only signal is a `warn!` on failure.
- **(Recommended)** Test the `prune_remote_plugin_cache_for_current_auth` failure path explicitly: mock the `/installed` endpoint to return 500 and assert that the cache is fully cleared (not retained). The fall-safe code path is currently untested per a quick scan.
- **(Recommended)** Defensive `plugin.installed.unwrap_or(true)` (or equivalent type-level Default) so a backend-schema regression that omits the field doesn't silently nuke every cached plugin.
- **(Optional)** Document the directory layout (`<codex_home>/plugins/cache/<marketplace>/<plugin_name_or_id>/<version>/.codex-plugin/`) somewhere reachable — this PR encodes it into both the production code and the test helpers but there's no central reference.

## Verdict: `merge-after-nits`

Correct security fix (account-switch cache leak is a real class of bug), correct fail-safe defaults (clear-on-fetch-failure), correct concurrency shape (`spawn_blocking` + at-most-one), with strong integration test coverage on the success path. The nits are observability (`tracing::info!`) and one missing failure-path test.

## What I learned

When the local cache is keyed on remote identity, the *cache invalidation point* is identity-change, not "cache TTL expired." This PR's three trigger points (logout / api-key login / chatgpt login completion) plus the `installed`-vs-`enabled` distinction are the right model. The other valid model would be lazy invalidation on first access — but for a plugin cache that's read by other subsystems on startup, eager invalidation at the identity-change boundary is the only safe choice.
