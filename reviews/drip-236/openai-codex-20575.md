# openai/codex#20575 — codex: migrate app-server thread data reads to ThreadStore

- **PR**: https://github.com/openai/codex/pull/20575
- **Head SHA**: `4160bf4d87303b97af882f7f5706759809cc6baf`
- **Size**: +161 / -310, 3 files
- **Verdict**: **merge-after-nits**

## Context

Third PR in the app-server `ThreadStore`-migration stack (alongside #20577 reviewed in drip-234 and #20576 reviewed in drip-235). Goal: replace direct rollout-file reads (`read_summary_from_rollout(rollout_path)` + `read_rollout_items_from_rollout(rollout_path)` + `latest_token_usage_turn_id_for_thread_path(...)` + `find_archived_thread_path_by_id_str` + `find_thread_name_by_id`) with `ThreadStore`-backed lookups that return a `StoredThread` containing summary, history, archive bit, and metadata in one round trip.

## What's right

**`bespoke_event_handling.rs:1235-1290` — rollback path collapses ~70 lines to ~30.**

Old code (replaced):
- Reject if `conversation.rollout_path()` is None.
- `read_summary_from_rollout(rollout_path, fallback_provider)` → `summary_to_thread(summary, &fallback_cwd)`.
- `read_rollout_items_from_rollout(rollout_path)` → `build_turns_from_rollout_items(&items)` → `thread.turns = ...`.
- `find_thread_name_by_id(codex_home, &conversation_id)` to attach name.
- Two distinct error-handling arms for the two file-reads.

New code:
- `conversation.read_thread(/*include_archived*/ true, /*include_history*/ true)` — single call returning `StoredThread`.
- `thread_from_stored_thread(stored, fallback_provider, &fallback_cwd)` returns `(Thread, Option<History>)` with name already attached.
- `populate_thread_turns_from_history(&mut thread, &history.items, /*active_turn*/ None)` rebuilds turns.
- One unified error-handling path.

The `thread_from_stored_thread` return signature `(Thread, Option<History>)` is the right shape: callers that need history get it in the second tuple slot without paying for it on the metadata-only path. The explicit `let Some(history) = history else { ... internal_error("did not include persisted history after rollback") }` at `:1268-1278` is the right fail-loud — rollback genuinely requires history; falling through with an empty turns list would silently corrupt the rollback view.

**`codex_message_processor.rs:3318-3414` — `update_thread_metadata` flips from direct state-db writes to `StoreThreadMetadataPatch`.**

Old code (deleted at `:3318-3500`):
- Acquire thread-list state-db permit.
- Resolve `state_db_ctx` via `loaded_thread.state_db()` or `get_state_db(&self.config)`.
- `ensure_thread_metadata_row_exists(thread_uuid, &state_db_ctx, loaded_thread.as_ref())` — three-arm helper that probed for row existence and synthesized one if missing (the helper itself is deleted at `:3380-3496`, ~120 lines gone).
- `state_db_ctx.update_thread_git_info(thread_uuid, sha, branch, origin_url)` — direct three-field UPDATE.
- `read_summary_from_state_db_context_by_thread_id` to reload.
- `summary_to_thread(summary, &self.config.cwd)` to convert.

New code:
- Build `StoreThreadMetadataPatch { git_info: Some(StoreGitInfoPatch { sha, branch, origin_url }), ..Default::default() }` and dispatch via `loaded_thread.update_thread_metadata(patch, /*include_archived*/ true)` if loaded, else `self.thread_store.update_thread_metadata(StoreUpdateThreadMetadataParams { thread_id, patch, include_archived: true })`.
- `thread_from_stored_thread(updated_thread, model_provider, &self.config.cwd)` to convert.

The dispatch split (loaded thread takes the live-thread path, unloaded thread goes through the store directly) is the right scope: `loaded_thread.update_thread_metadata(...)` presumably handles the "running thread needs to see its own metadata change" concern that the prior code reached for `state_db_ctx.update_thread_git_info` to handle (write through the loaded thread's own state-db handle). The 120-line `ensure_thread_metadata_row_exists` helper is gone because `ThreadStore::update_thread_metadata` now owns the upsert semantics.

**Test coverage referenced in PR body** (`thread_metadata_update`, `thread_fork_emits_restored_token_usage_before_next_turn`, `thread_resume_token_usage`, `thread_resume_emits_restored_token_usage_before_next_turn`, `thread_rollback_drops_last_turns_and_persists_to_rollout`, `review_start_with_detached_delivery_returns_new_thread_id`) covers the four critical contract surfaces: rollback drops-and-persists, resume emits-restored-token-usage, fork emits-restored-token-usage, and detached-review-fork returns-new-thread-id. The token-usage-replay tests are the most important regression pin because the PR body says "replay token usage from already-loaded store history instead of rollout paths" — the `latest_token_usage_turn_id_from_rollout_items(...)` path replaces `latest_token_usage_turn_id_for_thread_path(...)` (which is dropped from the imports at the top of `codex_message_processor.rs`), and the resume-emits-restored-token-usage test pins the wire shape of that boundary.

## Risks / nits

- **`include_archived: true` is the new default everywhere this PR adds a `read_thread` call** — at `:1240-1248` for rollback, at `:3367-3370` and `:3373-3376` for metadata-update. That's the correct semantic for rollback (you need to be able to roll back into an archived thread) and metadata-update (you need to be able to update metadata on an archived thread), but it's a behavior shift from the prior `find_thread_path_by_id_str` (non-archived only) → `find_archived_thread_path_by_id_str` (archived only) two-call pattern. Worth a one-line PR-body confirmation that "archived threads now appear in rollback/metadata-update without the prior explicit-archived-path lookup" is intentional and not a regression.
- **`ensure_thread_metadata_row_exists` deletion drops the row-not-found fallback path.** The prior helper synthesized a metadata row from `loaded_thread` if the state-db query returned None; the new code assumes `ThreadStore::update_thread_metadata` does the upsert internally. Worth a focused test arm pinning "update_thread_metadata against a thread whose state-db row was never created succeeds and creates the row" so the load-bearing upsert behavior on the store side has a regression pin in this PR's diff (currently it's an implicit dependency on `ThreadStore` test coverage in the other crate).
- **`#[cfg(test)] use codex_state::ThreadMetadataBuilder`** at `:373` (new) tells me `ThreadMetadataBuilder` is now only used by tests in this crate. If any production callers were using it, they're being silently removed; if not, the `#[cfg(test)]` gate is correct cleanup. Worth a quick `rg ThreadMetadataBuilder` confirmation in PR body.
- **`#[allow(unused_imports)]`-style suppressions on dropped imports.** The `use crate::codex_message_processor::read_rollout_items_from_rollout/read_summary_from_rollout/summary_to_thread` are flipped to `populate_thread_turns_from_history/thread_from_stored_thread`, and `use codex_core::find_thread_name_by_id` / `use codex_core::find_archived_thread_path_by_id_str` / `use codex_core::RolloutRecorder` are dropped. If any of those still has callers elsewhere in the crate not visible in this 3-file slice, you'll get unused-warning churn or compilation errors — worth a `cargo check -p codex-app-server` pre-merge confirmation. The `#[cfg(test)]` on `ThreadMetadataBuilder` and `use codex_thread_store::GitInfoPatch as StoreGitInfoPatch` suggest the import-level cleanup is tracked carefully, so this is likely fine.
- **`fallback_cwd = conversation.config_snapshot().await.cwd`** at `:1239` is acquired *before* the `read_thread(...)` call, so if the thread's stored cwd differs from the conversation's current cwd (rare but possible if the thread was forked across a `--cd` boundary), the fallback used by `thread_from_stored_thread` is the conversation's cwd. The prior code had the same shape so this isn't a regression, but a one-line comment that "fallback_cwd is the conversation's cwd, used only when stored thread doesn't carry its own cwd" would clarify the precedence rule for future readers.
- **`thread_from_stored_thread` returns a thread with `name` already attached** (PR body implies, the helper is in `codex_message_processor.rs` outside the visible slice). The metadata-update path at `:3408-3409` still calls `self.attach_thread_name(thread_uuid, &mut thread).await` afterward — is that redundant or is `thread_from_stored_thread`'s name attachment a no-op until `attach_thread_name` resolves the name from the watch manager? Worth a one-line comment or a redundancy elision.
- **Removed `_codex_home: &Path`** parameter at `:148` (was `codex_home: &Path` used by `find_thread_name_by_id` in the deleted rollback path). Underscore-prefixed unused-parameter is the right defensive pattern when the signature is API-stable across the wider codebase. If no other callers depend on this parameter, dropping it entirely (and updating the call site in `apply_bespoke_event_handling`) is cleaner; otherwise the underscore is correct.

## Verdict

**merge-after-nits.** Substantial cleanup (-149 net lines across 3 files) collapsing the dual rollout-file-read + state-db-direct-write paths into one `ThreadStore`-mediated path. The rollback hard-fail when history is missing, the `Default::default()`-spread `StoreThreadMetadataPatch`, and the explicit `include_archived: true` flags are the right primitives. Want PR-body confirmation on the archived-thread-default behavior shift, a regression test for the upsert-on-missing-row semantic, and a `cargo check` pass on the wider crate before merge.
