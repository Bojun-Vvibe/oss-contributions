# openai/codex #19456 — Add remote plugin uninstall API

- **PR:** https://github.com/openai/codex/pull/19456
- **Head SHA:** `fd8c33a5d66888324a0656ede3a440c3274c430d`
- **Files changed:** 10 — protocol schemas (4 JSON + 1 TS) and `protocol/v2.rs` (+42/−3), `core-plugins/src/remote.rs` (+39/−4), `app-server/src/codex_message_processor/plugins.rs` (+109/−6), `app-server/tests/suite/v2/plugin_uninstall.rs` (+248/−2), `app-server/README.md` (+1/−1).

## Summary

Extends `plugin/uninstall` from "local-only by `pluginId`" to also accept a remote-form `(remoteMarketplaceName, pluginName)` pair, which the app-server forwards as `POST /ps/plugins/{pluginId}/uninstall` to the ChatGPT plugin backend. The `PluginUninstallParams` Rust struct moves all three fields to `Option<String>` and the dispatcher pattern-matches on the tri-state, routing to either the existing local uninstall or the new `remote_plugin_uninstall` path.

## Line-level call-outs

- `codex-rs/app-server-protocol/src/protocol/v2.rs:4520-4528` — making all three fields `Option<String>` is the right shape for the union, but the JSON schema for `PluginUninstallParams` (e.g. `schema/json/v2/PluginUninstallParams.json:5-26`) drops the `required` array entirely. That means a request with `{}` passes schema validation and only fails at the Rust dispatcher (`plugins.rs:268-280`) with a runtime "requires either pluginId or…" error. Better: use a JSON Schema `oneOf` with two branches (`{required: ["pluginId"]}` vs `{required: ["remoteMarketplaceName","pluginName"]}`), so SDKs can statically reject malformed callers.
- `codex-rs/app-server/src/codex_message_processor/plugins.rs:258-287` — the tri-state `match`:
  ```rust
  (Some(plugin_id), None, None) => plugin_id,
  (None, Some(name), Some(plugin)) => { … remote path … }
  _ => invalid_request,
  ```
  The `_` catch-all conflates several distinct user errors into one message ("requires either pluginId or remoteMarketplaceName plus pluginName"). At minimum, distinguish "all three set" (caller bug — rejected) from "marketplace set but plugin name missing" (forgot a field) so logs are actionable.
- `plugins.rs:296-361` — `remote_plugin_uninstall` does the right things in the right order: load config → check `Plugins` *and* `RemotePlugin` features → re-validate `plugin_name` charset → fetch auth → call `uninstall_remote_plugin`. The double `is_valid_remote_plugin_id` check (once at the dispatcher `:261`, once at `:326`) is defensive duplication; fine to keep, but worth a `// validated again here in case dispatcher path changes` comment.
- `plugins.rs:357` — `clear_plugin_related_caches()` runs only on success. If the backend returned 200 with `enabled: false` but the local cache somehow gets out of sync on a partial-failure path (e.g. the response body is rejected by `uninstall_remote_plugin` because `id` mismatches), the cache stays stale. Consider clearing the cache regardless of remote outcome — the worst case is a redundant refetch.
- `app-server/tests/suite/v2/plugin_uninstall.rs:154-189` — the `_rejects_remote_marketplace_when_remote_plugin_is_disabled` test is exactly the right gate: confirms that flipping only the `Plugins` feature on (without `RemotePlugin`) keeps the remote path locked. The other test at `:454-498` mounts a wiremock server, asserts both `authorization` and `chatgpt-account-id` headers, and verifies the wire path `/backend-api/plugins/.../uninstall`. Solid coverage.
- `app-server/README.md:205` — doc copy is precise. One missed nuance: the doc says "uninstall a remote ChatGPT plugin … by forwarding the uninstall to the ChatGPT plugin backend"; doesn't mention the validation that the backend response must include the same `id` with `enabled: false`. Worth a sentence so consumers understand the contract isn't fire-and-forget.
- `protocol/v2.rs:10117-10146` — the additive serde test cases are good. Missing: a deserialize test that a `{}` payload round-trips as `(None, None, None)`, since that's the new "all-optional" wire shape; would lock in the schema's `required: []` decision deliberately.

## Verdict

**merge-after-nits**

## Rationale

The architectural choice (one endpoint, tagged-union parameters, dispatcher pattern-match) is the right approach for a backwards-compatible extension — better than introducing a parallel `plugin/uninstall_remote` request type that would force every client to learn two shapes. The Rust side is well-structured and well-tested. The two nits worth landing before merge: (1) tighten the JSON schema to a `oneOf` so static validators catch `{}` and `{pluginId, remoteMarketplaceName}` without a server round-trip, and (2) split the `_` catch-all error into the two distinct sub-cases so the failure mode is debuggable from the wire. Both are 5-line changes. Without them, the PR ships a working endpoint with a vague schema contract that will generate avoidable support traffic the first week it's enabled.
