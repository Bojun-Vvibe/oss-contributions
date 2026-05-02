# openai/codex PR #20825 — Read cached metadata for installed Git plugins

- Head SHA: `5d4d7e51e8fa1e173fb827b5f0db35f481a04681`
- URL: https://github.com/openai/codex/pull/20825
- Size: +248 / -4, 3 files (`codex-rs/app-server/tests/suite/v2/plugin_list.rs`,
  `codex-rs/core-plugins/src/manager.rs`,
  `codex-rs/core-plugins/src/manager_tests.rs`)
- Verdict: **merge-after-nits**

## What changes

When `PluginsManager::list_marketplaces_for_config` enumerates
marketplace plugins, it now reads the cached on-disk plugin manifest
(`<codex_home>/plugins/cache/<marketplace>/<plugin>/local/.codex-plugin/plugin.json`)
for installed Git-sourced plugins, instead of relying solely on the
`interface` field embedded in `marketplace.json`. The marketplace's
`category` is preserved (overrides the cached one) so you don't lose
curation when the plugin author ships a different category in their
own manifest. See `manager.rs:1183-1217`.

The two new tests
(`list_marketplaces_installed_git_source_reads_metadata_from_cache_without_cloning`
in manager_tests.rs:1986 and
`plugin_list_returns_installed_git_source_interface_from_cache` in
plugin_list.rs:1054) deliberately use a *missing remote URL* to prove
the read happens from cache and never touches the network — the test
also asserts `plugins/.marketplace-plugin-source-staging` does not get
created (manager_tests.rs:2089-2092).

## What looks good

- The `&& let ... && let ... && let ...` chain (manager.rs:1188-1193)
  correctly degrades to "use marketplace-supplied interface" if any
  step fails (no plugin id, no active root, no manifest file). Quiet
  fallback is the right behavior here — a missing cache file should
  not break `plugin_list`.
- `plugin_interface_with_marketplace_category(manifest.interface,
  marketplace_category)` (manager.rs:1196) keeps the marketplace as the
  source of truth for taxonomy while letting the plugin own everything
  else (display name, icon, screenshots, brand color). That's the
  correct ownership split.
- The "no network" assertion in the test (manager_tests.rs:2089-2092)
  is the right shape — it's an *observable* assertion (staging dir
  absence) rather than a mock-call count, so it stays meaningful even
  if the implementation is refactored.
- Asset path resolution
  (manager_tests.rs:2068-2078: `composer_icon`, `logo`, `screenshots`)
  correctly lands on absolute paths under the cache root, not relative
  paths from the user's cwd. That was a subtle bug class in earlier
  plugin work.

## Nits

1. The `&& let ... && let ...` (manager.rs:1188-1193) uses the
   experimental `let_chains` syntax. Confirm the workspace MSRV /
   `Cargo.toml` `rust-version` is bumped accordingly; if the repo is
   still on an older toolchain pin this will fail CI on a fresh
   checkout. The PR doesn't touch the toolchain file, so either
   `let_chains` is already stable on the pinned channel, or this needs
   an `#[allow]` / `#![feature]` somewhere.
2. The cached `category` is shadowed by the marketplace category
   *unconditionally* (manager.rs:1196 — `marketplace_category` always
   wins when present). What if the marketplace omits category? The
   helper presumably falls through to `manifest.interface.category`,
   but verify — the test only exercises the "marketplace has category,
   cache has different category" path (manager_tests.rs:2069 expects
   `"Developer Tools"`, not `"Cached Category"`). Add a second test
   for the "marketplace category absent → cache wins" case.
3. The cache root resolution goes through `self.store.active_plugin_root(&plugin_id)`
   (manager.rs:1191). When a Git plugin has multiple installed
   versions (sha-pinned vs ref-tracking), is `active_plugin_root` the
   right one to read from? The test uses the simple "local" subdir
   (`plugins/cache/debug/toolkit/local`) which doesn't exercise the
   sha-pinned variant.
4. `plugin_list.rs:1100-1109` duplicates a chunk of the manager-test
   fixture (the `marketplace.json` literal, the `plugin.json` literal,
   the `config.toml`). Worth extracting to a small `#[cfg(test)]`
   helper in a shared test-support crate to keep the two suites
   honest about the fixture they're testing.

## Risk

Low. The change is read-only at the cache layer, gated behind
`installed && matches!(source, MarketplacePluginSource::Git { .. })`,
and falls back silently on every error path. Worst case: a corrupted
cached manifest causes `load_plugin_manifest` to return `None` and the
old marketplace-supplied interface is used — same behavior as before.

The only real exposure is correctness of the `category` priority rule
(see nit 2) — if the marketplace and the cached manifest disagree, the
PR enshrines "marketplace wins" without explicit user-facing
documentation. Worth a one-liner in the marketplace authoring docs.
