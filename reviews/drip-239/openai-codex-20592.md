# openai/codex#20592 — Fix thread-store archived live fallback

- **PR**: https://github.com/openai/codex/pull/20592
- **Author**: dylan-hurd-oai
- **Head SHA**: `5020581e701e4c57b01895d9801cfc0b024cec42`
- **Files**: `codex-rs/thread-store/src/local/read_thread.rs` (+5 / -0)
- **Verdict**: **merge-as-is**

## Context

`read_thread` in `thread-store/src/local/read_thread.rs` resolves a `thread_id` to a rollout file via a path-resolution chain that, when the active rollout for a thread isn't found, falls back to the archived rollout location. That fallback is correct for `include_archived=true` callers (they explicitly want archived data), but for active-only reads (`include_archived=false`) it silently returns archived data dressed up as live — meaning the caller asking "give me the *live* rollout for this thread" can get back data with `archived_at: Some(...)` and have no signal that what they got isn't what they asked for.

The narrow case the PR description calls out — "when a live writer was resumed from an archived source" — is the failure mode where the live rollout *legitimately* doesn't exist at the active-rollout path because the writer was bootstrapped off the archived file, so the path resolver lands on the archived file as a fallback and returns it. Active-only reads then get an `archived_at: Some(...)` thread back.

## What's right

- **The check is at the right layer.** `read_thread.rs:74-79` adds the guard *after* `read_thread_from_rollout_path(...)` resolves and parses the file (so the `archived_at` field is populated by the parser, not inferred from the path), but *before* `attach_history_if_requested(...)` does the (potentially expensive) history-attachment work. A read that's going to be rejected as archived now fails fast with no I/O wasted on history.
- **The error variant is the right shape.** `ThreadStoreError::InvalidRequest { message: format!("thread {thread_id} is archived") }` (`:76-78`) tells the caller exactly what went wrong with enough detail to act on (the thread ID is in the message), without leaking implementation details about the rollout-path-resolution chain that would couple the caller to the storage layout.
- **The condition is the precise predicate.** `!params.include_archived && thread.archived_at.is_some()` — the negation is on the *caller's intent* (`include_archived`) and the positive check is on the *file's actual state* (`archived_at.is_some()`). This is the right direction: don't infer "this is archived data" from path heuristics or fallback flags; check the parsed file's own marker.
- **The validation is the kind that matters.** The PR body cites `cargo test -p codex-thread-store load_history_uses_live_writer_rollout_path_for_archived_source -- --nocapture (3 consecutive passes)` — three consecutive runs is the right discipline for any test that lives next to a path-resolution chain (which can have ordering-sensitivity bugs) plus the full-package `cargo test -p codex-thread-store` run for regression. The named test (`load_history_uses_live_writer_rollout_path_for_archived_source`) suggests the regression is being pinned at the contract level: "live-writer-resumed-from-archived → load history → must come from live writer's rollout path, not the archived source".

## Risks / nits

- **The `format!("thread {thread_id} is archived")` message** is functional but doesn't differentiate between "you asked for active-only and this thread is permanently archived" (a stable rejection) vs. "you asked for active-only and the live rollout briefly fell back to archived because of a writer-resume race" (a transient-ish state). Probably fine for now since both should result in the same caller behavior (treat as archived and either pass `include_archived=true` or skip), but a future "list archived threads matching filter" surface might want a structured field rather than a string.
- **No assertion that `archived_at` itself is consistent with the path resolution** (e.g. if the file resolver landed on the archived path but the file's `archived_at` is `None`, the read currently *succeeds* — silently returning what is presumably stale or corrupted data). Not blocking, since that's a separate bug class (path-vs-content drift) and probably doesn't happen in practice, but worth a `debug_assert!` or a parser-side check.

## Verdict

**merge-as-is.** Five-line guard at the right point in the read path, correct predicate, correct error shape, three consecutive test runs in validation. The nits are about message structure and a parallel consistency check that don't matter for this PR's scope.
