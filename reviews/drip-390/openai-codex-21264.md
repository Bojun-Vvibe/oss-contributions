# Review: openai/codex#21264 — Move thread name edits to ThreadStore

- Head SHA: `e661af17eb66a355ad8e94d34f60575c1521e969`
- Files: 4 (+41 / -51)
- Verdict: **merge-after-nits**

## Summary

Routes live thread renames through `ThreadStore` metadata updates and reads resumed thread names from store metadata, with legacy local fallback preserved inside the store. Net subtractive (+41/-51) — collapses the dual code path through `state_db_ctx` + `find_thread_name_by_id` filesystem fallback into a single store-mediated read.

## Specific evidence

- **Replaces `title_from_state_db` + `find_thread_name_by_id` filesystem fallback** at `codex-rs/app-server/src/request_processors/thread_processor.rs:3567-3593` (deleted) with direct `self.thread_store.read_thread(StoreReadThreadParams { thread_id, include_archived: true, include_history: false })` at `:2852-2860` — single source of truth, no more two-tier lookup with separate failure modes.
- **Title trim + non-empty + non-redundant guard** at `:2861-2862`:
  ```rust
  && let Some(title) = stored_thread.name.as_deref().map(str::trim)
  && !title.is_empty()
  && stored_thread.preview.trim() != title
  ```
  Preserves the same three-condition guard as the deleted `distinct_title()` (the `title == first_user_message → suppress` rule) — correct, the preview field is the resume-time replacement for `first_user_message`.
- **`include_history: false`** at `:2858` — important: this is called on every `attach_thread_name` invocation, so paying for full history hydration would scale poorly with thread length. Worth a comment naming this hot path.
- **`include_archived: true`** at `:2857` — correct, since archived threads can still be opened in the resume picker and need a title.
- **Subtractive imports** at `request_processors.rs:269` (deleted `find_thread_name_by_id`) and `core/src/session/mod.rs:39` (deleted `crate::rollout::find_thread_name_by_id`) — confirms the legacy filesystem walk is no longer reachable from `app-server`. Worth one `rg 'find_thread_name_by_id' codex-rs/` to confirm the symbol is gone or only retained as a `pub(crate)` fallback inside the store itself.
- **`ResumeThreadParams` adds `ReadThreadParams` import** at `core/src/session/mod.rs:135` — used for the new store-mediated read path inside `Session`. Net subtractive there too (24+/15-).

## Nits

1. **No new test** added in this PR for the title-attach path itself. The PR is small and the rule is the same three-condition guard, but a focused unit test in `thread_processor_tests.rs` covering (a) empty-name returns no title, (b) name == preview returns no title, (c) name with surrounding whitespace gets trimmed, would harden the migration.
2. **Performance**: `read_thread` is now called for every `attach_thread_name`. If the store implementation does any I/O (sqlite stat, lock acquisition), this is a per-thread overhead on every resume-picker render. Worth a maintainer confirm that `read_thread` with `include_history: false` is O(1) metadata lookup in all existing `ThreadStore` impls (in-memory + sqlite + remote).
3. **The `preview.trim() != title` check**: `preview` is built from the first user message at thread-create time. If the user later renames the thread to literally match the original first message, the rename is silently dropped. Probably correct (the existing `distinct_title()` had the same behavior) but worth a comment.
4. The PR body is two lines — the *why* (single source of truth, removes filesystem-walk fallback in app-server) is worth one paragraph in the commit message for archaeologists.

Migration is sound, net subtractive, semantics preserved.
