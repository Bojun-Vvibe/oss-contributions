# openai/codex PR #21187 — app-server: refresh live threads from latest config snapshot

- URL: https://github.com/openai/codex/pull/21187
- Head SHA: `ad56d24edaf0db1037200b4f125e8a17fdc3a1ea`
- Author: jif-oai
- Size: +157 / -26 (4 files: app-server config processor, codex_thread, session/mod, session/tests)

## Verdict
`merge-after-nits`

## Rationale

The PR fixes a real staleness bug in the right place. Before this change, `ConfigRequestProcessor`
at `config_processor.rs:386` reacted to a config mutation by sending each live thread `Op::ReloadUserConfig`,
which re-parsed `config.toml` from disk inside `Session::reload_user_config_layer()`. That path
(`session/mod.rs:1424-1486` pre-diff) only re-read the user TOML layer and ignored everything the
app-server had already materialized in its in-memory config snapshot (effective layer stack, derived
`tool_suggest`, hook/plugin/skill state from non-file sources). The new code at
`config_processor.rs:386-396` reads each thread's current `cwd`, calls `self.load_latest_config(Some(cwd))`
to rebuild a snapshot using the same path the rest of the app-server uses, then pushes it via
`thread.refresh_runtime_config(next_config).await`. This eliminates a class of "settings only take
effect after restart" bugs and removes a `core` filesystem access by routing the rebuild through the
host that already owns the snapshot.

The split between session-static and runtime-refreshable fields is well-defined and pinned by tests.
`Session::refresh_runtime_config()` at `session/mod.rs:1394-1424` only swaps `config_layer_stack` and
`tool_suggest` into a clone of `original_config_do_not_use`, then rebuilds `hooks` from the new
config and republishes if and only if `Arc::ptr_eq` confirms no concurrent refresh raced ahead — the
same in-flight protection pattern the file-based path used. The legacy `reload_user_config_layer()`
is correctly preserved as a delegation to `refresh_runtime_config()` (no duplicated hook-rebuild
logic), with an explicit doc-comment at `:1463-1465` saying "prefer `refresh_runtime_config()` when
the host can already provide a materialized config snapshot". Two new tests pin both halves of the
contract: `refresh_runtime_config_refreshes_hooks` confirms that adding a `SessionStart` hook to
`config.toml` makes `session.hooks().preview_session_start(&request)` go from `[]` to length 1, and
`refresh_runtime_config_updates_runtime_refreshable_fields_and_keeps_session_static_settings` pins
that `model` and `notify` are *not* swapped.

Nits to address before / shortly after merge, none blocking: (1) `config_processor.rs:387` does
`thread.config_snapshot().await.cwd.to_path_buf()` then re-acquires the thread on the next line via
`load_latest_config` — the cwd is captured under one lock and used outside it, which is correct but
worth a one-line comment that "stale cwd here only affects which layer files we read, and the
`Arc::ptr_eq` guard in `Session::refresh_runtime_config` means a concurrent thread cwd change won't
publish a wrong snapshot". (2) The `tracing::warn!` at `:391-394` swallows the `err.message` but
loses `thread_id` from context — adding `thread_id` to the log line would materially help debugging
when one thread's refresh fails out of N. (3) `refresh_runtime_config()` is `pub async fn` on
`CodexThread` but the comment at `codex_thread.rs:460-462` should explicitly list which fields are
"runtime-refreshable" rather than just saying "Session-static settings such as model and permissions
remain unchanged" — the implementation pins exactly two (`config_layer_stack`, `tool_suggest`) plus
derived `hooks`/`plugins`/`skills`, and that list will drift unless documented.
