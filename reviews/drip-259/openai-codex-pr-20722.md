# openai/codex PR #20722 — Remove remote plugin uninstall prefix gate

- PR: https://github.com/openai/codex/pull/20722
- Head SHA: `5d9d97c3dda4996c501767afd558d9c7f09ec9f2`
- Author: @xli-oai
- Size: +79 / -21

## Summary

Removes a hardcoded allow-list of remote plugin ID prefixes (`plugins~`, `plugins_`, `app_`, `asdk_app_`, `connector_`) from the app-server's plugin uninstall path. Local plugin IDs of the form `plugin@marketplace` continue to be parsed by `codex_plugin::PluginId::parse`; anything that fails that local-shape parse now flows directly into the remote uninstall path, where the generic `is_valid_remote_plugin_id` safety check (rejecting empty, spaces, slashes, etc.) is still applied before the URL/cache hit.

Rationale: the backend plugin-service owns the remote-ID contract, so codex shouldn't be the one rejecting newly-introduced ID families.

## Specific references from the diff

- `codex-rs/app-server/src/codex_message_processor/plugins.rs:776-783` — replaces the dual-gate `parse(...).is_err() && !is_valid_remote_uninstall_plugin_id(...)` rejection with the simpler "not a local ID → try remote" branch. The `if codex_plugin::PluginId::parse(&plugin_id).is_err()` check now unconditionally calls `self.remote_plugin_uninstall_response(plugin_id).await`.
- `codex-rs/app-server/src/codex_message_processor/plugins.rs:903-910` — deletes the entire `is_valid_remote_uninstall_plugin_id` helper that encoded the prefix allow-list.
- `codex-rs/app-server/tests/suite/v2/plugin_uninstall.rs:401-470` — adds `plugin_uninstall_accepts_remote_plugin_id_without_known_prefix` covering a `newremote_…` ID with a mocked backend.
- `tests/suite/v2/plugin_uninstall.rs:531, 589, 623` — three existing rejection tests updated to expect `"invalid remote plugin id"` instead of `"invalid plugin id"`, since the local-shape error path no longer fires for empty / space-bearing IDs.

## Verdict: `merge-after-nits`

The change is well-scoped, the new test exercises the unblocked path against a mocked backend, and the safety net (`is_valid_remote_plugin_id`) is preserved. The only thing that gives me pause is the silent expansion of "what counts as a remote ID" — anything not matching `plugin@marketplace` is now sent to the backend.

## Nits / concerns

1. **Failure mode for typos changed.** Before: a typo like `"plugin-foo"` was rejected synchronously with a clear message at the app-server. After: it's sent to the backend uninstall endpoint, which returns a 404 / "plugin not found" that bubbles back as a generic error. Worth noting in release notes — operators debugging "uninstall does nothing" will now have to look one network hop deeper. A clarifying log line at `plugins.rs:782` ("treating %s as remote ID; routing to backend") would help support.
2. **Renamed test names without renaming the rejection-message constant.** `plugin_uninstall_rejects_invalid_plugin_id_before_remote_path` → `plugin_uninstall_rejects_remote_plugin_id_with_spaces_before_network_call` is a substring of the same scenario. Fine, but the matching error string `"invalid remote plugin id"` is now hardcoded in three different test assertions. Pull it into a `const EXPECTED_INVALID_REMOTE_MSG: &str` so the next message tweak doesn't require touching three tests.
3. **No coverage for the empty-string regression risk.** `is_valid_remote_plugin_id("")` presumably returns false (the existing `plugin_uninstall_rejects_empty_remote_plugin_id` test still passes), but the *path* through the new code is: `PluginId::parse("")` errors → falls into `remote_plugin_uninstall_response("")` → which then rejects. Worth an inline assertion that the empty-string case still produces the same error code (-32600) before merging, to confirm we didn't make `""` look like a valid backend call.
4. **Description says `git diff --check` was run** but the diff includes no whitespace-only changes; the four `validation` checks listed are good practice — keep doing this on plugin-routing PRs.
