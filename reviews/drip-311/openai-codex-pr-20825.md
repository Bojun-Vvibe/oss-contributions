# openai/codex #20825 — Read cached metadata for installed Git plugins

- PR: https://github.com/openai/codex/pull/20825
- Author: xli-oai
- Head SHA: `5d4d7e51e8fa1e173fb827b5f0db35f481a04681`
- Updated: 2026-05-02T22:03:15Z

## Summary
When a marketplace plugin sourced from git is already installed (i.e. its working copy lives under `<codex_home>/plugins/cache/<marketplace>/<plugin>/local/`), `PluginsManager.list_marketplaces` should surface the *cached* `plugin.json` interface (displayName, icons, etc.) rather than only the marketplace.json's coarse interface block. Previously, hitting an unreachable git remote meant the gateway would refuse to enumerate cached interface fields, so the TUI couldn't render icons for installed plugins offline. Production change in `codex-rs/core-plugins/src/manager.rs:1185-1215`. Two new integration tests: `plugin_list_returns_installed_git_source_interface_from_cache` (`codex-rs/app-server/tests/suite/v2/plugin_list.rs:1054-1170`) and `list_marketplaces_installed_git_source_reads_metadata_from_cache_without_cloning` (`codex-rs/core-plugins/src/manager_tests.rs:1985+`).

## Observations
- `codex-rs/core-plugins/src/manager.rs:1186-1203`: the new logic gates the cache lookup on `installed && matches!(&plugin.source, MarketplacePluginSource::Git { .. })`. Good — non-git sources (local, registry) keep their existing path. The chained `let-else` (`if installed && matches!(...) && let Ok(plugin_id) = ... && let Some(plugin_root) = ... && let Some(manifest) = load_plugin_manifest(...)`) requires let-chains; confirm the MSRV in `Cargo.toml` allows let-chains in stable (Rust 1.65+ for if-let chains, 1.88+ for let-chains in `&&`). If the project still pins an older edition, this won't compile. Worth a CI sanity check.
- `codex-rs/core-plugins/src/manager.rs:1196-1201`: `plugin_interface_with_marketplace_category(manifest.interface, marketplace_category)` — the marketplace category overrides the manifest category, but other manifest fields (displayName, icons, brandColor) win. That's the right precedence: marketplace authors curate categories, plugin authors own their own branding.
- `codex-rs/app-server/tests/suite/v2/plugin_list.rs:1054-1170`: the test deliberately uses a non-existent git remote URL (`missing-remote-plugin-repo`) and pre-populates the cache at `<codex_home>/plugins/cache/debug/toolkit/local/.codex-plugin/plugin.json`. This exactly models the "offline / remote went 404" scenario. Asserts the cached interface fields surface (displayName="Toolkit", brandColor="#3B82F6", composerIcon path canonicalized correctly, marketplace category "Developer Tools" overriding the manifest's "Cached Category").
- `codex-rs/app-server/tests/suite/v2/plugin_list.rs:1166-1170`: `composer_icon` and `logo` assertions canonicalize the cached plugin root via `std::fs::canonicalize`. On macOS this resolves `/private/var/folders/...` symlinks; the canonicalize call here matches what production code presumably does. Good defensive pattern.
- `codex-rs/core-plugins/src/manager_tests.rs:1985+`: the unit-level twin test asserts the same behavior at the manager API surface. Two-tier coverage (unit + app-server integration) is appropriate for plugin manifest resolution since regressions are easy to introduce.
- `manager.rs:1207-1213`: `id: plugin_key` (no longer `plugin_key.clone()`) — the `installed`/`enabled` lookups happen before the move via the new local bindings on lines 1186-1187. Looks correct, but worth a `cargo build` to confirm no use-after-move warnings.
- Edge case the PR doesn't test: what if the cached `plugin.json` is malformed? `load_plugin_manifest` presumably returns `None` and we silently fall back to the marketplace interface. That's the right default but a one-line trace log would help debugging.

## Verdict
`merge-after-nits`
