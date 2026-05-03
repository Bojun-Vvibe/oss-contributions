# openai/codex PR #20825 — Read cached metadata for installed Git plugins

- Author: xli-oai
- Head SHA: `5d4d7e51e8fa1e173fb827b5f0db35f481a04681`
- Diff: +248 / -4 across 3 files
- Files: `codex-rs/app-server/tests/suite/v2/plugin_list.rs`, `codex-rs/core-plugins/src/manager.rs`, `codex-rs/core-plugins/src/manager_tests.rs`

## Observations

1. **`manager.rs:1183-1199` is the load-bearing change**: when a plugin is `installed` AND its source is `MarketplacePluginSource::Git { .. }`, `list_marketplaces_for_config` now reads `plugin.json` from `self.store.active_plugin_root(&plugin_id)` and replaces the marketplace's `interface` with the cached manifest's interface (preserving the marketplace's `category` via `plugin_interface_with_marketplace_category`). The chained `if let … && let … && let …` is rust-2024 let-chains — readable, idiomatic, no allocation in the cold path.
2. **Why this matters**: previously a Git plugin's `displayName`/`composerIcon`/`logo`/`brandColor` came from the marketplace JSON only. After installation those assets live on disk under the plugin cache, so reading from cache means the UI no longer needs the upstream marketplace repo to render rich metadata. The integration test at `plugin_list.rs:341` deliberately constructs `missing_remote_repo` (a URL pointing to a directory that never exists) to prove the read works *without* hitting the remote.
3. **`plugin_list.rs:431-445` asserts canonicalized absolute paths**: `composer_icon` and `logo` resolve to `cached_plugin_root/assets/{icon,logo}.png` after `std::fs::canonicalize`. This is the right shape, but be aware: canonicalization requires the file to exist on disk. The test creates `.codex-plugin/plugin.json` but doesn't actually create `assets/icon.png`/`logo.png` files — confirm whether `AbsolutePathBuf::try_from` here implicitly canonicalizes (it shouldn't, normally) or whether the manifest loader does. If manifest loading canonicalizes, the test will silently pass on this machine and fail in CI on a different filesystem layout. Worth a quick `ls` in the test setup to make sure the assertion shape is what runs in production.
4. **`manager.rs:1190-1196` preserves `marketplace_category`**: `interface.as_ref().and_then(|i| i.category.clone())` is captured *before* overwriting `interface`, then re-injected via `plugin_interface_with_marketplace_category`. Good — this is the field where the marketplace must win over the cached manifest (the cached manifest's `"category": "Cached Category"` is ignored in favor of the marketplace's `"Developer Tools"`, asserted at `manager_tests.rs:593`).
5. **Non-Git sources unaffected**: the `matches!(&plugin.source, MarketplacePluginSource::Git { .. })` guard ensures local/file sources keep their existing behavior. Good blast-radius control.
6. **Test coverage is thorough**: both the unit test (`manager_tests.rs:1985-…`) and the integration test through the JSON-RPC surface (`plugin_list.rs:336-451`) exercise the same path. The duplication is justified — they catch different classes of regression.

## Verdict: `merge-after-nits`

The change is well-scoped and the test scaffolding is impressive (the "missing remote URL" trick is a clean way to prove cache-only reads). Only nit: double-check that the `composer_icon`/`logo` path comparison doesn't depend on canonicalization-of-nonexistent-files behavior; if it does, create the asset files in the test setup so the assertion is portable.
