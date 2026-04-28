# openai/codex #19966 — Require remote plugin detail before uninstall

- PR: https://github.com/openai/codex/pull/19966
- Author: xli-oai
- Head SHA: `8124e7bb9bd7`
- Diff: 52+/124- across 2 files (`codex-rs/app-server/tests/suite/v2/plugin_uninstall.rs`, `codex-rs/core-plugins/src/remote.rs`)

## Verdict: merge-after-nits

## Rationale

- **Right shape: pre-fetch detail, fail-fast, then POST.** The pre-fix flow at `core-plugins/src/remote.rs:548-576` POSTed the uninstall first, *then* tried to fetch detail (for cache-namespace cleanup), and on detail-fetch failure it just `warn!`-logged and skipped the named-cache removal. That's a "backend committed, local cache leaked" divergence on every transient detail-fetch error. The fix at `:528-533` flips the order: `fetch_plugin_detail(...)` is now the gate before the POST, so on detail failure the backend never sees the uninstall request and local cache stays intact. Symmetric, no divergence.
- **Helper `fetch_remote_plugin_detail_by_id` at `:438-457` correctly deleted.** It was the indirection that made the pre-fetch optional. Now the call site uses `fetch_plugin_detail` directly with `include_download_urls=false` and extracts `marketplace_name` + `plugin_name` upfront. Cache cleanup function signature flips from `Option<RemotePluginDetail>` to `String, String, String` (marketplace, name, legacy id) — types now reflect the new "always required" contract.
- **Test rename `plugin_uninstall_posts_even_when_remote_detail_fetch_fails` → `plugin_uninstall_rejects_before_post_when_remote_detail_fetch_fails` at `tests/suite/v2/plugin_uninstall.rs:327` is the load-bearing rename.** The previous behavior was named explicitly in the test name; the new behavior is named explicitly. The assertions flip: `expected_count: 1` → `0` for the POST, new `expected_count: 1` for the GET, error code `-32600` with `"remote plugin catalog request"` substring, and crucially `assert!(legacy_remote_plugin_cache_root.exists())` (was `!exists()`) — pinning the "cache untouched on failure" invariant.
- **Mock `mount_empty_remote_installed_plugins` deletion at `:551-567` is correct.** The previous flow called the `/installed` endpoint as a fallback path; new flow only calls `/ps/plugins/{id}` (detail). Removing the unused mount is the right cleanup — leaving it would have been a "what is this for?" trap for the next reader.

## Nits / follow-ups

- The error message substring assertion `err.error.message.contains("remote plugin catalog request")` at `:367` is brittle — if the upstream `RemotePluginCatalogError` formatter ever rewords, this test breaks silently with a misleading error. Pin the error variant rather than the message text if `RemotePluginCatalogError` is `Debug`/`PartialEq`.
- No test for the **success** path of detail-then-POST ordering (i.e., GET happens before POST in the wire trace). The current tests pin "POST didn't happen on detail failure" but not "GET-then-POST happened in order on detail success". A `wait_for_remote_plugin_request_count(GET, ..., 1)` assertion *before* `wait_for_remote_plugin_request_count(POST, ..., 1)` in `plugin_uninstall_writes_remote_plugin_to_cloud_when_remote_plugin_enabled` would lock that invariant.
- Behavior change is user-visible: previously a flaky detail endpoint would still let users uninstall (with a noisy cache-cleanup warning); now it blocks them. This is the right call but worth noting in release notes — users on intermittent networks will see a failure mode they didn't see before. The PR body could call this out explicitly.
- `legacy_plugin_id: String` is still threaded through `remove_remote_plugin_cache` — preserved for the legacy-cache-root cleanup path. Worth a one-line comment at the call site that this is intentional dual cleanup (legacy + named), not a redundant param.

## What I learned

"Backend-then-local" two-phase commits where the local phase can fail without rolling back the backend phase are a classic divergence shape — the previous code had it (`POST` then `fetch_detail` for cache cleanup, with the second step being best-effort). The fix flips to "verify-then-mutate": fetch detail first, fail fast, only POST on success. This is a structural fix, not a behavior tweak — it changes the durability contract from "backend wins, local is best-effort" to "both succeed together or neither". The `Option<RemotePluginDetail>` → `(String, String)` signature change is the type-system advertising the new contract. The lesson generalizes: any time you see a `match { Ok => Some, Err => warn!+None }` pattern guarding the second half of a two-phase mutation, ask whether the second phase should be a *gate* on the first rather than a *consequence*.
