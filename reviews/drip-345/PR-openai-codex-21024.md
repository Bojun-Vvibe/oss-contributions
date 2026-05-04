# openai/codex#21024 — fix flaky test load_history_uses_live_writer_rollout_path_for_archive

- PR ref: `openai/codex#21024` (https://github.com/openai/codex/pull/21024)
- Head SHA: `b60e850708d128f627bd875fc1f82130595e54c9`
- Title: fix flaky test load_history_uses_live_writer_rollout_path_for_archive…
- Verdict: **merge-after-nits**

## Review

The root cause is real: when `apply_rollout_items` flushes a thread whose rollout path
sits under `ARCHIVED_SESSIONS_SUBDIR`, the prior code wrote the rollout but never
populated `archived_at` in the state DB, so a follow-up read that gated on archived
status would race with the SQLite row's `NULL` default. The fix at
`codex-rs/rollout/src/state_db.rs:539-541` initializes `builder.archived_at` from the
`updated_at_override`-or-now timestamp before `apply_rollout_items` runs, then the
explicit `mark_archived` call at `state_db.rs:553-560` ensures the marker survives even
when the builder pathway didn't persist it. That's belt-and-braces but defensible since
`apply_rollout_items` has multiple write paths that have historically dropped fields.

The detection helper `rollout_path_is_archived` at `state_db.rs:566-569` walks the
`Path::components()` looking for `ARCHIVED_SESSIONS_SUBDIR` as an `OsStr` match. Correct
on both Unix and Windows since `Components` already normalizes separators. The matching
guard added to `read_thread.rs:34-43` and `read_thread.rs:85-91` closes the read side:
even if the SQLite metadata is somehow stale, an archived rollout path now produces an
explicit `InvalidRequest` instead of silently returning the thread.

Nits:
1. `state_db.rs:548-550` early-returns on the inner `apply_rollout_items` error, which
   means the new `mark_archived` step never runs in the failure path — fine, but worth a
   debug log that says "skipping archived marker because apply failed" so operators
   reading logs can correlate.
2. The two near-identical `rollout_path_is_archived` checks in `read_thread.rs` (lines
   34 and 85) take slightly different signatures (`store.config.codex_home.as_path()` vs
   the local helper). Worth a follow-up to consolidate; not blocking.
3. The new assertion at `local/mod.rs:667-678` proves `archived_at` is populated, but
   there's no negative test that confirms a *non-archived* flush leaves `archived_at`
   as `None`. Easy to add and would lock the contract.
