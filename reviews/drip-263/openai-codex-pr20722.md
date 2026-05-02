# openai/codex PR #20722 — Remove remote plugin uninstall prefix gate

- **Head SHA**: `17c9fdea9dc222d60a2916b928fc85f996ea76bf`
- **Scope**: `codex-rs/app-server/src/codex_message_processor/plugins.rs` + tests

## Summary

Drops the bespoke `is_valid_remote_uninstall_plugin_id` helper and routes the uninstall handler through the same `is_valid_remote_plugin_id` check used elsewhere. See `plugins.rs:776-780` (validation collapsed) and `plugins.rs:907-915` (helper removed). Tests in `tests/suite/v2/plugin_uninstall.rs` are updated to expect the new error message `"invalid remote plugin id"`.

## Comments

- Reduces drift between install and uninstall validation paths — good. Two divergent allowlists for the same identifier shape was a footgun.
- Error message is now less informative: previously listed the accepted prefixes (`plugins~`, `plugins_`, `app_`, `asdk_app_`, `connector_`); now says only `"invalid remote plugin id"`. Consider keeping the prefix enumeration in the error text — it's the kind of message client devs read in stack traces.
- One test was renamed from `plugin_uninstall_rejects_invalid_plugin_id_before_remote_path` to `..._with_spaces_before_network_call` (line 457). Renaming a regression test to describe the input rather than the contract is a minor smell — the rename narrows future readers' understanding of intent.
- No behavior change for clients that already used valid prefixed IDs; failure mode is now "ambiguous error" rather than "wrong allowlist."

## Verdict

`merge-after-nits` — preserve the prefix list in the error message for better DX.
