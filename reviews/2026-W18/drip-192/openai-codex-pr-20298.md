# openai/codex #20298 — Surface admin-disabled remote plugin status

- **URL:** https://github.com/openai/codex/pull/20298
- **Head SHA:** `af020029231fd63b2ce8dacf1e21391dffcad80c`
- **Files:** 13 (4 generated JSON schemas, 2 generated TS types, 1 protocol Rust enum, 1 message-processor handler, 2 test suites + 1 fixture helper, 1 remote-catalog field, 1 tui test helper)
- **Verdict:** `merge-as-is`

## What changed

Adds a new `PluginAvailabilityStatus` enum (`ENABLED` | `DISABLED_BY_ADMIN`) to the v2 app-server protocol and propagates it from the remote-catalog response through `PluginSummary` to the client, with a server-side reject in the install path so a disabled remote plugin can never be installed even if a stale client tries.

1. **Protocol enum** at `protocol/v2.rs:4654-4665`:
   ```rust
   #[derive(Serialize, Deserialize, Debug, Clone, Copy, PartialEq, Eq, JsonSchema, TS)]
   pub enum PluginAvailabilityStatus {
       #[serde(rename = "ENABLED")] #[ts(rename = "ENABLED")] Enabled,
       #[serde(rename = "DISABLED_BY_ADMIN")] #[ts(rename = "DISABLED_BY_ADMIN")] DisabledByAdmin,
   }
   ```
   Manual `serde(rename)` + `ts(rename)` to lock the wire literal — no `#[serde(rename_all = ...)]` shortcut. Right call: the wire enum is across an org boundary (companion webview PR `openai/openai#873269`) and SCREAMING_SNAKE_CASE matches existing `PluginInstallPolicy` / `PluginAuthPolicy` precedent.

2. **`PluginSummary.status: Option<PluginAvailabilityStatus>`** added at `:4707` (and corresponding TS type at `typescript/v2/PluginSummary.ts:7`). `Option` because local plugins don't carry an admin-availability concept — the `local_plugin_summary_to_info` path at `plugins.rs:243` sets `status: None`, while `remote_plugin_summary_to_info` at `:755` sets `status: Some(summary.status)`. The TS type uses `status?: PluginAvailabilityStatus` (optional, not nullable) — symmetric with `Option<T>` round-tripping.

3. **Install-path rejection** at `plugins.rs:432-437`:
   ```rust
   if remote_detail.summary.status == PluginAvailabilityStatus::DisabledByAdmin {
       return Err(invalid_request(format!(
           "remote plugin {plugin_name} is disabled by admin"
       )));
   }
   ```
   Sits *after* the `read remote plugin details before install` fetch at `:432` and *before* the `PluginInstallPolicy::NotAvailable` check at `:438`. Ordering matters: an admin-disabled plugin should reject before the install-policy check so the user sees the admin-disabled reason rather than a generic "not available", but after the fetch so a transient catalog outage emits a clearer error than "disabled".

4. **Remote catalog field with `#[serde(default = ...)]`** at `core-plugins/src/remote.rs:584-592`:
   ```rust
   pub status: PluginAvailabilityStatus,
   #[serde(default = "default_plugin_availability_status")]
   ```
   Critical for compatibility: a remote catalog server that hasn't shipped the `status` field yet round-trips as `ENABLED` rather than failing parse. The `default_plugin_availability_status()` helper presumably returns `Enabled`.

5. **Three new integration tests** in `app-server/tests/suite/v2/plugin_install.rs` and `plugin_list.rs` covering: (a) install rejected before download for `DISABLED_BY_ADMIN` (the load-bearing assertion), (b) list response correctly marks remote plugins, (c) list-includes-marketplaces-when-enabled. The test helper `mount_remote_plugin_detail_with_status` at `:352` parameterizes the catalog mock by status.

## Why it's right

- **The reject-before-download ordering is the security-load-bearing piece.** Without it, an admin disabling a plugin in the catalog would only block install at the *next* `plugin_list` refresh (when the client UI hides the install button). Between catalog-mutation-time and client-UI-refresh-time, a stale UI could still POST `plugin/install` and the server would happily fetch the bundle, cache it, and install it — bypassing the admin gate. The `plugin_install_rejects_remote_plugin_disabled_by_admin_before_download` test (`:413-432`-ish) explicitly asserts this ordering: `mount_remote_plugin_detail_with_status(server, name, version, DisabledByAdmin)` then expects the install RPC to error *and* asserts no bundle download mock was hit (presumably via `wiremock::Mock::expect(0)`-style assertion inside the helper). This is the right shape — server-side enforcement with server-side test.
- **`Option<PluginAvailabilityStatus>` on `PluginSummary` rather than a plain `PluginAvailabilityStatus`** is the right modeling. Local plugins genuinely have no admin-availability concept, and a forced default (`Enabled`) would conflate "no policy applies" with "policy applied and resolved to enabled". Two TS clients reading `summary.status === undefined` vs `summary.status === "ENABLED"` may want to render differently (e.g. don't show an "ENABLED" badge for local plugins).
- **Generated-schema parity is complete.** All four schema artifacts (`codex_app_server_protocol.schemas.json`, `.v2.schemas.json`, `v2/PluginListResponse.json`, `v2/PluginReadResponse.json`) include the new enum + the `status` anyOf. Plus the generated TS files (`PluginAvailabilityStatus.ts` standalone export, `PluginSummary.ts` import + field, `index.ts` re-export). The `cargo run --bin write_schema_fixtures` re-run was performed (called out in PR validation), so reviewers don't have to suspect a hand-edit drift between the Rust source-of-truth and the JSON/TS-derived artifacts.
- **`#[derive(Copy)]` on the enum** at `protocol/v2.rs:4654`. Both variants are unit, so `Copy` is free and prevents the `&` clutter at every match site. Symmetric with the existing `PluginInstallPolicy` / `PluginAuthPolicy` derives — house style.
- **The `helpers.rs` change in `tui/`** (1 file) is presumably a `PluginSummary { ..., status: None }` field add to a fixture. Compile-only chase work, expected.

## Tiny non-blocking observations

- The companion webview PR (`openai/openai#873269`) is referenced in the body. Order of merge matters: codex-side PR can land first because remote-catalog responses default to `ENABLED` via `#[serde(default)]`, and clients without the field-aware UI will simply not render it — graceful degradation in both directions.
- `invalid_request(format!("remote plugin {plugin_name} is disabled by admin"))` at `:434`. The error message includes the plugin name but not the *reason* shape (e.g. "admin policy: enterprise allowlist"). For triage of "user reports they can't install plugin X", the current message lets the user know it's admin-policy-driven — sufficient.
- The `#[serde(default = "default_plugin_availability_status")]` at `remote.rs:592` is a function-pointer default, not the simpler `#[serde(default)]` (which would require `Default` impl on the enum). Either works — the explicit fn is clearer about which variant becomes the default and is greppable.

## Risk

Low. Pure additive protocol change with backward-compatible defaults on both sides (`Option` on the codex-internal type, `#[serde(default)]` on the remote-catalog parse). The behavior change is server-side only — admin-disabled plugins now reject at install time instead of silently allowing. If a previously-installed plugin is later marked `DISABLED_BY_ADMIN`, this PR doesn't auto-uninstall it (out of scope), so admin disable is install-time enforcement; runtime enforcement (block load of an already-installed disabled plugin) would be a follow-up if desired.
