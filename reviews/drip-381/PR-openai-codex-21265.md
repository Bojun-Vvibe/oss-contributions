# openai/codex PR #21265 — Route ThreadManager rollout path reads through thread store

- URL: https://github.com/openai/codex/pull/21265
- Head SHA: `3a00dd6c7b3dc7cea124400c9c8a8c38e73e99c9`
- Size: ~150 added / ~10 removed (core + thread-store + tests)

## Summary

Eliminates direct `RolloutRecorder::get_rollout_history(&path)` reads from `ThreadManager` for the resume / fork-from-rollout-path code paths, routing them through the configured `ThreadStore` via a new `read_thread_by_rollout_path(...)` call. Adds a new private helper `initial_history_from_rollout_path` at `core/src/thread_manager.rs:839-856`, swaps three call sites (`resume_thread_from_rollout`, `:730` second resume variant, `fork_thread`), and patches `InMemoryThreadStore` to actually return the persisted `rollout_path` mapping it had been silently dropping.

## Specific findings

- `thread_manager.rs:665, :734, :828` — three `RolloutRecorder::get_rollout_history(&...).await?` call sites all replaced with `self.initial_history_from_rollout_path(...)`. Consistent.
- `thread_manager.rs:839-856` — new helper clones the requested `rollout_path` once before consuming it, calls `thread_store.read_thread_by_rollout_path(ReadThreadByRolloutPathParams { rollout_path, include_archived: true, include_history: true })`, then forwards to `stored_thread_to_initial_history(stored_thread, Some(requested_rollout_path))`. `include_archived: true` is the right call for resume/fork — an archived thread should still be resumable from its on-disk rollout file.
- `thread_manager.rs:1330-1346` — `stored_thread_to_initial_history` constructs `InitialHistory::Resumed(ResumedHistory { conversation_id: thread_id, history: history.items, rollout_path: rollout_path.or(stored_thread.rollout_path) })`. The `or` precedence (caller-provided `Some(requested)` wins) is correct: if a user explicitly resumed against a path, that path stays authoritative even if the stored thread has its own canonical path.
- `thread_manager.rs:1348-1354` — error mapper translates `ThreadStoreError::ThreadNotFound { thread_id }` to `CodexErr::ThreadNotFound(thread_id)` and `InvalidRequest { message }` to `CodexErr::InvalidRequest(message)`. **Concern:** the previous `RolloutRecorder::get_rollout_history` had its own error taxonomy (file-not-found vs parse error vs IO error). Routing everything through `ThreadStoreError` collapses those into `CodexErr::Fatal(format!(...))` for the catch-all branch. If a downstream consumer (TUI, IDE) was matching on the old error variants for "rollout file missing → offer to create new thread" UX, that UX silently breaks. Worth a maintainer ack.
- `thread_manager.rs:1331-1335` — `let history = stored_thread.history.ok_or_else(|| CodexErr::Fatal(format!("thread {thread_id} did not include persisted history")))?` — note that calling `read_thread_by_rollout_path` with `include_history: true` should always populate this; the `Fatal` is a defensive panic-equivalent. Could be a `debug_assert!` instead of a runtime error message users could see, but acceptable.
- `thread-store/src/in_memory.rs:259-264` — fixes a real correctness bug: `stored_thread_from_state` was hardcoding `rollout_path: None` even when `state.rollout_paths` actually mapped a path to that `thread_id`. New code does a `state.rollout_paths.iter().find_map(|(path, mapped_thread_id)| (*mapped_thread_id == thread_id).then(|| path.clone()))`. Linear scan is fine for in-memory test stores. **Nit:** if multiple paths map to the same thread_id (e.g. a fork created a second mapping), `find_map` returns the first encountered, which is order-of-insertion-dependent in `HashMap`/`IndexMap`. If `rollout_paths` is a `HashMap`, this is non-deterministic. Worth checking the type.
- `thread_manager_tests.rs:624-728` — new 100+ line integration test `rollout_path_resume_and_fork_read_history_through_thread_store` exercises the full flow: start thread → shutdown → seed an `InitialHistory::Resumed` with a synthetic rollout path → `resume_thread_from_rollout` → `fork_thread` → assert `in_memory_store.calls().await.read_thread_by_rollout_path == 2` (one for resume, one for fork). Pins the contract that both paths route through the store, not the recorder.
- The dropped `use crate::rollout::RolloutRecorder;` at `:10` confirms the module no longer references `RolloutRecorder` directly in this path.

## Notes

- This PR pairs naturally with #21264 (move thread name edits to ThreadStore) and #21260 (already in drip-379, which moved thread naming) — together they're consolidating rollout state ownership behind the ThreadStore trait. Reviewer should understand it as part of that sequence.
- No CHANGELOG entry — but it's an internal refactor with no external API surface change *except* the error-taxonomy concern above.
- Test does not cover the "rollout path exists on disk but thread store has no mapping" failure mode (the `ThreadNotFound` branch). Would harden the error contract.

## Verdict

`merge-after-nits`
