# openai/codex PR #20825 — Read cached metadata for installed Git plugins

- **Repo:** openai/codex
- **PR:** #20825
- **Head SHA:** `5d4d7e51e8fa1e173fb827b5f0db35f481a04681`
- **Author:** xli-oai
- **Title:** Read cached metadata for installed Git plugins
- **Diff size:** +248 / -4 across 3 files
- **Drip:** drip-294

## Files changed

- `codex-rs/core-plugins/src/manager.rs` (+22/-4) — at marketplace plugin enumeration (`list_marketplaces` aggregation, around line 1183), when a plugin's source is `MarketplacePluginSource::Git { .. }` AND `installed_plugins.contains(&plugin_key)` is true, load the locally cached `plugin.json` via `load_plugin_manifest(plugin_root.as_path())` and overlay the cached `interface` (preserving the marketplace's `category` via `plugin_interface_with_marketplace_category`).
- `codex-rs/core-plugins/src/manager_tests.rs` (+109/-0) — new unit test `list_marketplaces_installed_git_source_reads_metadata_from_cache_without_cloning` that pre-populates a cache dir with a `plugin.json` and asserts the response uses the cached interface fields.
- `codex-rs/app-server/tests/suite/v2/plugin_list.rs` (+117/-0) — integration test `plugin_list_returns_installed_git_source_interface_from_cache` exercising the full request/response loop with a deliberately unreachable git URL.

## Specific observations

- `manager.rs:1186-1201` — clean use of `&&`-chain `if let` (Rust 2024 let-chains). Reads top-to-bottom: installed → is git source → can build PluginId → cache root exists → manifest loads. Good progressive narrowing; on any failure the original (marketplace-supplied) `plugin.interface` is kept, which is the right fallback.
- `manager.rs:1196-1199` — `marketplace_category` is captured *before* the overlay so the marketplace's category wins over the cached manifest's category. The test at `app-server/tests/suite/v2/plugin_list.rs:1145` exercises exactly this (cache says `"Cached Category"`, marketplace says `"Developer Tools"`, result is `"Developer Tools"`). Good — but the precedence rule deserves a one-line comment in `manager.rs` so future readers don't "fix" the perceived bug.
- `manager.rs:1208-1213` — moving from `id: plugin_key.clone()` to `id: plugin_key,` and reordering `installed`/`enabled` to use the locally bound vars is a nice cleanup that the diff sneaks in. Worth calling out in the PR description.
- The integration test deliberately uses a missing remote (`missing-remote-plugin-repo` URL points at a non-existent dir) to prove the cache path doesn't trigger a clone. That's exactly the right shape of test — it would fail loudly if a regression re-introduced network IO.
- No behavior change for non-git sources or for git plugins that aren't yet installed. Both fall through to the existing `plugin.interface` value, so the blast radius is tightly scoped.
- The cached `composer_icon` / `logo` fields are resolved via `std::fs::canonicalize(&cached_plugin_root)` in the test — confirms the manifest loader produces absolute paths. Worth an explicit assertion comment in `manager.rs` that downstream serializers expect absolute paths.

## Verdict: `merge-as-is`

This is a small, well-scoped fix with both unit and integration coverage that pins the no-network-IO contract. The cleanup of `installed`/`enabled` binding ordering is a bonus. Ship.
