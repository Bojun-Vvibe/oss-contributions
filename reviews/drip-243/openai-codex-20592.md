# Review: openai/codex #20592 — Fix thread-store archived live fallback

- **PR**: https://github.com/openai/codex/pull/20592
- **Author**: dylan-hurd-oai (Dylan Hurd)
- **Head SHA**: `5020581e701e4c57b01895d9801cfc0b024cec42`
- **Base**: `main`
- **Files**: `codex-rs/thread-store/src/local/read_thread.rs` (+5/-0)
- **Verdict**: **merge-as-is**

## Reasoning

Five-line correctness patch closing a real fallback bug in `read_thread`
at `thread-store/src/local/read_thread.rs:74-79`. The story:

- `read_thread` resolves a thread to a rollout path. A *live writer*
  resumed from an archived source can land at a rollout file whose
  `archived_at` field is populated.
- The pre-fix code unconditionally called
  `read_thread_from_rollout_path(...)` and then attached history,
  silently returning archived content even when the caller passed
  `params.include_archived = false`. That's a contract violation:
  active-only reads should not see archived data.
- The fix adds a guard immediately after the rollout read:
  `if !params.include_archived && thread.archived_at.is_some() { return
  Err(ThreadStoreError::InvalidRequest { … }); }`.

Why this is the right shape:

- The check fires *after* the rollout read and *before*
  `attach_history_if_requested`, which is correct — `archived_at` lives
  on the thread metadata, not in the history blob, so the read has to
  happen first to know whether to bail. Cost is one disk read on the
  bad-fallback path, which is acceptable for an error path.
- The error variant is `InvalidRequest`, which matches the semantic
  shape: from the caller's perspective, asking for a non-archived
  thread by an ID that resolves to an archived rollout *is* an invalid
  request, not a missing-thread case (which has its own variant
  upstream of this guard).
- The error message (`"thread {thread_id} is archived"`) is specific
  enough to be debuggable from logs without leaking storage internals.
- The author validated with three consecutive runs of
  `load_history_uses_live_writer_rollout_path_for_archived_source`
  plus the broader `cargo test -p codex-thread-store`, plus `just fmt`
  and `just fix`. The fact that the targeted test name explicitly
  encodes the live-writer-from-archived-source scenario suggests the
  test was authored alongside the fix, which is the right move.

## Suggested follow-ups

- Optional: if `ThreadStoreError` has a dedicated `Archived` variant
  elsewhere in the crate, prefer that over `InvalidRequest` for type
  precision. Not blocking — `InvalidRequest` is semantically defensible.
- Optional: add a `tracing::debug!` on the error path so operators
  can see live-writer/archived-source mismatches in logs without
  having to grep error strings. Cheap insurance against the case
  where this triggers from an unexpected resume path.
