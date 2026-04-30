# openai/codex PR #20309 — Move plugin manager out of core

- PR: https://github.com/openai/codex/pull/20309
- Head SHA: `7f106fae542b5db812efcc2267ee9645165ab7bb`
- Files touched: 49 files. Notable: `core-plugins/src/manager.rs` (+177/-501, the bulk migration), `core/src/plugins_manager_tests.rs` (+137/-481, paired migration), deletions of `core/src/plugins/startup_sync.rs` (-100), `core/src/plugins/startup_sync_tests.rs` (-90), plus protocol additions of `PluginAvailabilityStatus` enum across 4 schema files + `app-server-protocol/src/protocol/v2.rs:4657-4672` + `PluginSummary` field at `:4679-4682`.

## Specific citations

- New protocol type `PluginAvailabilityStatus { Enabled, DisabledByAdmin }` at `v2.rs:4657-4666` with `#[serde(rename = "ENABLED" | "DISABLED_BY_ADMIN")]` and `ts(rename = ...)` mirrors. Generated TS at `schema/typescript/v2/PluginAvailabilityStatus.ts` and threaded into `PluginSummary` at `schema/typescript/v2/PluginSummary.ts:7`.
- `PluginSummary.status: Option<PluginAvailabilityStatus>` field at `v2.rs:4679-4681` is `#[serde(default, skip_serializing_if = "Option::is_none")]` and `#[ts(optional)]` — non-breaking on the wire.
- `codex_message_processor.rs:6294-6300, 6379-6390`: API surface change — `effective_skill_roots_for_layer_stack` and `plugins_for_layer_stack` now take destructured feature flags (`config.features.enabled(Feature::Plugins)`, `Feature::RemotePlugin`, `Feature::PluginHooks`) instead of a `&Config` reference. Decouples core from `Config`.
- `plugins.rs:42-49`: `maybe_start_plugin_list_background_tasks` similarly takes scalar args (`plugins_enabled`, `remote_plugin_enabled`, `chatgpt_base_url.clone()`, `auth.clone()`, ...) instead of `&Config`.
- `plugins.rs:84, 261`: every `PluginSummary` constructor now sets `status: None` (no admin-disable backend yet — feature is plumbed but not wired).
- `plugins.rs:378-385`: New behavior — after a successful `install_plugin`, the handler calls `ConfigEditsBuilder::new(config.codex_home).set_plugin_enabled(&installed_plugin_id, true).apply()` to *persist* the install as enabled. Previously the plugin had to be enabled separately.
- Cargo.lock at `:2503-2532` confirms `core-plugins` now depends on `codex-analytics`, `codex-tools`, `tracing-subscriber`, `tracing-test` — three of those (analytics/tools/tracing-test) are new edges that should be reviewed for layering correctness (analytics inside the plugin manager could feed back into tracing-test mocks at integration time).
- `core/src/plugins/mod.rs` shrinks +14/-23 (lighter facade); `startup_sync.rs` and `startup_sync_tests.rs` deleted entirely (presumably moved into `core-plugins`).
- `manager.rs` is **net -324 lines** (501 deletions / 177 additions) — significant code consolidation from the move.

## Verdict: needs-discussion

## Concerns

1. **49-file PR is too large for atomic review**. The PR conflates: (a) protocol additive `PluginAvailabilityStatus` field, (b) `core` → `core-plugins` migration of `manager.rs`/`plugins_manager_tests.rs`, (c) API-surface refactor from `&Config` to scalar feature flags across 8+ call sites, (d) install-time auto-enable persistence at `plugins.rs:378-385`, and (e) deletion of the entire `startup_sync` module. Each is independently reviewable; bundling them costs reviewers the ability to bisect a regression.
2. **`status: None` everywhere** at `plugins.rs:84, 261` and the new generated schema fields ship a contract surface that has no backing implementation in this PR. Either land a follow-up PR queued in the description that wires up the actual `DISABLED_BY_ADMIN` source, or add a `// TODO(#xxxx): populate from policy engine` so the next reader doesn't think the field is dead.
3. **The install-time auto-enable at `plugins.rs:378-385`** is a real behaviour change buried in a "move" PR. Previously, a fresh install required a separate `set_plugin_enabled(true)` ConfigEdit; now it's atomic. This is the right shape but the semantics change (e.g. existing tests that asserted `enabled=false` post-install would now fail) deserves call-out in the PR body and a CHANGELOG entry.
4. **Layer drift risk**: `core-plugins` now depends on `codex-analytics` and `codex-tools` (Cargo.lock `:2507, 2518`). If `codex-tools` already depended on `codex-core` (likely — tools execute against core) this could create a cycle through a transitive edge. Confirm `cargo tree -p codex-core-plugins` doesn't show `codex-core` in its dependency closure.
5. **Test migration rebalance**: `core/src/plugins_manager_tests.rs` lost 481 lines and `core-plugins/src/discoverable_tests.rs` gained 219, `manager.rs` gained 177. Net -85 test lines plus the deleted `startup_sync_tests.rs` (-90) is **-175 test LOC overall**. Confirm explicitly that no test was *dropped* in the move (the cleanest verification is grep for `#[test]`/`#[tokio::test]` count in the deleted files vs added files).
6. **Empty PR body** — for a 49-file PR with this much surface change (new protocol enum, install-time semantic change, module move, three cross-cutting API refactors), the PR body should at minimum list the four sub-changes and confirm `cargo test --workspace` is green. Currently the only signal is the title.
7. **Recommended split**: (i) protocol-only `PluginAvailabilityStatus` enum + field, (ii) `core` → `core-plugins` move with mechanical re-exports, (iii) `&Config` → scalar feature flags refactor, (iv) install-time auto-enable behaviour change with its own regression test. Four PRs land in roughly the same wall-clock as this one but each is reviewable in 10 minutes.
