# openai/codex#20577 — codex: use store history for core review forks

- **PR**: https://github.com/openai/codex/pull/20577
- **Head SHA**: `5f7faf80e6317b740f21b65d00b9756888762a3f`
- **Size**: +23 / -25, 3 files
- **Verdict**: **merge-after-nits**

## Context

Codex previously had two paths to load the parent thread's rollout/history when forking for sub-agent spawn or guardian-review: (1) the live `Session::ensure_rollout_materialized` + `flush_rollout` + `RolloutRecorder::get_rollout_history(rollout_path)` "snapshot the JSONL on disk" path, and (2) `find_thread_path_by_id_str(...)` for archived parents. This PR is part of the broader migration to a `ThreadStore`-backed unified history reader (see related #20575 / #20576) — collapses both code paths to use the in-process `ThreadStore` whenever the live thread is available, falling back to file-based lookup only for archived parents.

## Design analysis

### `agent/control.rs` (sub-agent fork — `:366-396`)

New shape:
```rust
let parent_thread = state.get_thread(parent_thread_id).await.ok();
let mut forked_rollout_items = if let Some(parent_thread) = parent_thread.as_ref() {
    parent_thread.flush_rollout().await?;
    parent_thread.load_history(/*include_archived*/ true).await
        .map_err(|err| CodexErr::Fatal(format!(
            "failed to load parent thread history for fork {parent_thread_id}: {err}"
        )))?
        .items
} else {
    let rollout_path = find_thread_path_by_id_str(...).await?
        .ok_or_else(|| CodexErr::Fatal(...))?;
    RolloutRecorder::get_rollout_history(&rollout_path).await?
        .get_rollout_items()
};
```

Two real changes vs. the old code:
1. **Removed `ensure_rollout_materialized()` for live parents.** The new `parent_thread.load_history(true)` reads from `ThreadStore` which already has the materialized state — the on-disk JSONL no longer needs to be the canonical source for live forks.
2. **`flush_rollout` is now called on the thread, not on `parent_thread.codex.session`.** Same effect (drains queued writes), but the API is now thread-scoped. This is only called in the live-thread arm; the archived arm doesn't need it because the JSONL is already final.

The `if let Some` → `else` rewrite also collapses the prior `parent_thread.as_ref().and_then(...).or(...).ok_or_else(...)` chain into two clear arms. Easier to reason about.

### `guardian/review_session.rs` (`:775-779`)

Old: `session.flush_rollout() → session.current_rollout_path() → RolloutRecorder::get_rollout_history(path) → history.get_rollout_items()`.
New:
```rust
session.flush_rollout().await?;
let live_thread = session.live_thread_for_persistence("guardian review fork")?;
let history = live_thread.load_history(/*include_archived*/ true).await?;
Ok(Some(history.items))
```

Same migration. Removes the now-unused `RolloutRecorder` import at `:38`. The `live_thread_for_persistence` accessor takes a context string ("guardian review fork") for error messages — clean.

### `rollout.rs` (`:51-55`)

Adds `#[allow(unused_imports)]` on the `pub use codex_rollout::RolloutRecorder` re-export. Honest acknowledgment that the migration leaves the symbol re-exported for backward-compat consumers but not used internally anymore.

## What's right

- **Single source of truth.** Before, live forks read the materialized JSONL on disk that was just flushed for the snapshot — round-tripping through the FS for data the in-process store already had. After, live forks read directly from `ThreadStore`, archived forks fall back to file-based.
- **Error message carries `parent_thread_id`** in the new fork-side path, which the old `?` did not. Easier to debug a fork that fails because the store load broke.
- **Symmetric treatment in both fork sites** — sub-agent spawn and guardian-review use the same pattern (`flush_rollout` + `load_history(true)`), so future readers don't have to wonder which path is canonical.
- **`#[allow(unused_imports)]`** rather than removing the re-export means downstream crates that still import `RolloutRecorder` via `core::rollout::recorder::RolloutRecorder` don't break. Conservative.

## Nits / discussion

- **Behavior change risk on `include_archived: true`.** The old `RolloutRecorder::get_rollout_history(rollout_path)` read whatever was in the file at flush time. The new `load_history(true)` includes archived items from the store. For a long-lived thread that's been compacted/archived items pruned from the live in-memory state but retained in store-archive, the fork now contains *more* than it would have before. Worth a one-line PR-body note confirming whether `include_archived: true` is the right choice for both the sub-agent fork (which then runs `truncate_rollout_to_last_n_fork_turns(...)` so the extra items get clipped anyway) and the guardian-review fork (which uses the items verbatim).
- **No fallback in the live-thread arm.** If `state.get_thread(...).await` returns `Some` but `load_history(true)` then fails (e.g. transient store error), the code returns `Fatal` without falling back to the file-based path. Old code would have hit the file path because `parent_thread.as_ref().and_then(parent_thread.rollout_path())` was the source. Probably correct (store should be authoritative) but a brief comment explaining "no FS fallback for live threads — store is canonical" would help the next maintainer.
- The removed `Session::ensure_rollout_materialized` call site is one of presumably several. If this is the last caller, the method itself can be dropped in a follow-up.
- No new tests in this PR. The companion #20575 / #20576 likely carry the integration coverage; worth confirming in the PR body.

## Verdict

**merge-after-nits** — clean migration to the unified `ThreadStore` path, removes a redundant materialize-flush-then-read cycle, error messages improved. Wants a one-line PR-body note on the `include_archived: true` semantics and ideally a comment at the live-thread arm explaining the no-FS-fallback decision.
