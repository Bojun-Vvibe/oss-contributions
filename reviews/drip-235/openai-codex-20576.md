# openai/codex#20576 — codex: add store-backed live thread metadata helpers

- **PR**: https://github.com/openai/codex/pull/20576
- **Head SHA**: `fb8978919e6288d47ad3d283a2eb137c2a765f4b`
- **Size**: +238 / -7, 3 files
- **Verdict**: **merge-after-nits**

## Context

Mid-stack PR in the `wiltzius/codex/thread-store-migration-stack` series (sibling of the already-reviewed #20577 and #20575). Adds two helpers (`read_thread`, `update_thread_metadata`) onto `CodexThread` that route through `LiveThread::live_thread_for_persistence` so callers no longer need direct `LocalThreadStore` references, and lifts the `git_info` field of `ThreadMetadataPatch` from "not implemented in this slice" to fully supported in the local store.

## What's right

**`codex_thread.rs:401-432` — two new public methods.**

`CodexThread::read_thread(include_archived, include_history)` and `update_thread_metadata(patch, include_archived)` both follow the same pattern: grab `live_thread_for_persistence` from `self.codex.session`, map the `anyhow::Error` to `ThreadStoreError::Internal { message: err.to_string() }`, then delegate to the new `LiveThread` methods. The `"read thread"` and `"update thread metadata"` labels passed to `live_thread_for_persistence` will surface in the materialization error trail if the thread isn't yet persisted, which is the right diagnostic.

**`live_thread.rs:144-186` — symmetric `LiveThread` methods.**

The two helpers (`read_thread` and `update_metadata`) construct `ReadThreadParams { thread_id: self.thread_id, ... }` / `UpdateThreadMetadataParams { thread_id: self.thread_id, patch, include_archived }` and forward to the underlying `ThreadStore`. The `thread_id` is owned by `LiveThread`, so the helper API is "what to do, not which thread" which is the right shape for a live-thread reference.

**`local/update_thread_metadata.rs` — git_info implementation.**

The previous slice rejected `params.patch.git_info.is_some()` outright. New code:

1. **Field-count guard at `:33-41`** generalizes the prior pairwise rejection (`name && memory_mode`) to a count of all three fields (`name + memory_mode + git_info`), still permitting at most one per call. Cleanly handles the new field without adding a third pairwise arm.
2. **Pre-flush at `:46-48`** — `if live_writer::rollout_path(store, thread_id).await.is_ok() { live_writer::persist_thread(store, thread_id).await?; }`. Lazy/in-memory-only threads get materialized to disk *before* the metadata patch is applied, so the subsequent `read_thread` after the patch sees a persisted state. This is the load-bearing fix called out in the PR title ("materialize live local rollouts before metadata updates so persisted metadata exists for lazy threads").
3. **`apply_thread_git_info` at `:98-126`** routes the three sub-fields (`sha`, `branch`, `origin_url`) through `state_db.update_thread_git_info(thread_id, sha, branch, origin_url)`. The `Option<Option<String>>` shape of `GitInfoPatch` (outer `Option` = "patch field touched", inner `Option<String>` = "set or clear") is correctly forwarded as `value.as_ref().map(|v| v.as_deref())` — `Some(Some(s))` becomes `Some(Some(&str))` (set), `Some(None)` becomes `Some(None)` (clear), `None` becomes `None` (skip).
4. **Disappeared-row check at `:118-124`** — `if updated { Ok(()) } else { Err(...) }` correctly distinguishes "the row was found and updated" from "the row vanished between flush and update", returning a meaningful internal error rather than silently no-op'ing.
5. **Test coverage at `:362-490` (visible portion).** `update_thread_metadata_sets_git_info` pins the all-three-fields-set path with the expected `Some("abc123")`/`Some("main")`/`Some("https://github.com/openai/codex")` round-trip; the second test (`update_thread_metadata_partially_updates_git_info`, partially visible at the diff cutoff) presumably pins the `Option<Option<>>` clear-vs-set semantics, which is exactly the brittle spot.

## Risks / nits

- **Pre-flush is unconditional on rollout-path success.** The `if live_writer::rollout_path(...).await.is_ok()` predicate at `:46` calls `persist_thread` whenever the path is resolvable, including for the `name` and `memory_mode` paths that previously didn't pre-flush. Behavior change for those paths is "they now also pre-flush before patching". For `name`, this is consistent with how `apply_thread_name` reads the `resolved_rollout_path` — the prior code would have failed at `resolve_rollout_path` for never-flushed threads anyway, so this just moves the failure to a deterministic flush step. Worth a one-line comment at `:46-48` explaining "pre-flush so all three patch arms see the same on-disk state".
- **`git_info` is captured into a local before any patch arm runs (`let git_info = params.patch.git_info;` at `:50`)** then re-applied at `:71-73`. The reorder means git_info is the *last* patch applied, after `name` and `memory_mode`. Since the field-count guard ensures at most one is `Some`, this ordering is moot today, but if the guard is later relaxed to allow combined patches, the ordering becomes load-bearing — worth a comment, or hoist the patch handling into a more obviously-ordered pipeline.
- **`StoredThread` re-export** — the new `pub use codex_thread_store::StoredThread` and `ThreadMetadataPatch` imports at `codex_thread.rs:34-35` are part of the public surface for downstream `core` crate consumers. Worth a one-line note in the changelog or PR body that `CodexThread::read_thread` returns `StoredThread` (not `Thread`) since the discrepancy could confuse callers wiring this into agent-side code.
- **No test that the pre-flush actually runs for a never-materialized thread.** All three visible test arms call `write_session_file` before invoking the metadata update — i.e. the rollout file already exists on disk. A test that constructs a thread purely in-memory (no `write_session_file`), then invokes `update_thread_metadata`, and asserts the rollout file was created as a side-effect would directly pin the load-bearing change in the PR title.
- **`StoredThread` returned from `read_thread(include_history=false)` semantics** — does the `history` field come back as `None`, an empty `Vec`, or the field is omitted in the serialization? Worth a one-line doc-comment on the new `LiveThread::read_thread` method since `include_history=false` is the cheap-fast-path the helper exists to enable.

## Verdict

**merge-after-nits.** The git-info implementation is correctly shaped (Option<Option<>> clear-vs-set semantics handled), the pre-flush is the right fix for the lazy-thread metadata-update case, and the helpers on `CodexThread` close the gap that #20577 needed in the `core` crate. Address the comment-clarity nits (pre-flush rationale at `:46`, ordering rationale on `git_info`-last) and add the never-materialized-thread test before merge.
