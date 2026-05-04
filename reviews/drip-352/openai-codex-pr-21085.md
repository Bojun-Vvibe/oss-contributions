# openai/codex #21085 — [codex] Use app/list for TUI app catalog

- URL: https://github.com/openai/codex/pull/21085
- Head SHA: `1fca28782e057bf2b06c0349787506098e407df8`
- Author: canvrno-oai
- Size: +244 / -104 across 7 files

## Comments

1. `codex-rs/app-server/src/request_processors/apps_processor.rs:44-58` — Good cleanup: the config branch now correctly uses `thread_config.cwd.to_path_buf()` as fallback for the thread-scoped path, which fixes the prior bug where workspace-relative provider config was missed. Worth a regression test with a thread whose cwd differs from process cwd.
2. `apps_processor.rs:96-122` — The new `Ok/Err` split around `apps_list_response` correctly avoids sending the force-refetch notification on error. However the `outgoing.send_result(request_id, response)` is awaited in both arms; if it fails, `should_force_refetch_after_response` won't fire — which is the desired ordering, but please add a comment so the next reader doesn't "fix" it.
3. `apps_processor.rs:340-345` (`AppsListTaskResult`) — Struct has 3 fields all consumed once at the call site. Consider returning a tuple or making the fields `pub(super)` to keep the helper-type weight low.
4. `apps_processor.rs:404-446` (`send_force_refetched_app_list_updated_notification`) — The `tokio::join!` runs both refetches in parallel which is right. But on the `Err` branches you `return` without notifying the client that the force-refresh failed — the UI will just keep stale data forever. At minimum log at `warn!` (you do) and consider sending a generic "refresh-failed" notification so the TUI can re-trigger.
5. `apps_processor.rs:443-446` — The `if data != previous_data` equality check on `Vec<AppInfo>` requires `AppInfo: PartialEq`; ensure that derive is in place and that ordering is stable (otherwise spurious notifications).
6. `codex-rs/tui/Cargo.toml:32` — Removing `codex-chatgpt` dep is the actual TUI-side switch to `app/list`. Verify `cargo build -p codex-tui --no-default-features` still passes; also that this doesn't break any feature gate downstream.
7. `codex-rs/tui/src/chatwidget/tests/popups_and_settings.rs` — Diff truncated at 200 lines; please ensure the test for the new `app/list` driven catalog actually asserts pagination + refetch ordering.

## Verdict

`merge-after-nits`

## Reasoning

The refactor is well-scoped: it moves TUI app discovery to the canonical `app/list` request/response pair and wires through a "force refetch after first response" path so the user sees fast initial render with a follow-up refresh. The branching for thread-vs-no-thread config is now more readable. Main risks are the silent-on-error behaviour of the deferred notification and the fact that `AppInfo` equality must be stable to avoid notification storms. Nothing in the diff looks structurally wrong; tighten error visibility and add a unit test that exercises `should_force_refetch_after_response = true` end-to-end.
