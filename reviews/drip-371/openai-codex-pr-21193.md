# openai/codex PR #21193 — Avoid phase2 agent for empty memory housekeeping

- URL: https://github.com/openai/codex/pull/21193
- Head SHA: `f7456567ce63b195a714e38316cc1ad0ecf32d5f`
- Author: dylan-hurd-oai (Dylan Hurd) — DRAFT
- Files: `codex-rs/memories/write/src/{phase2.rs,startup_tests.rs}` (+39/-7)

## Assessment

This is a tight cost-saving change: when a phase-2 memory-write run finds that the only workspace changes are housekeeping (empty raw memories file, rollout-summary additions, or extension-resource deletions), it commits the workspace baseline reset and short-circuits to `succeeded_empty_input_housekeeping` without spawning a consolidation agent. The new branch at `phase2.rs:155-191` is precisely scoped: it only fires when `raw_memories.is_empty()` AND every change in `workspace_diff.changes` matches one of three cheap patterns — the raw-memories filename itself, anything under `ROLLOUT_SUMMARIES_SUBDIR/`, or a deletion under `EXTENSIONS_SUBDIR/`. The `change.status == GitBaselineChangeStatus::Deleted` guard on the extensions prefix is the key correctness anchor: an *added* or *modified* extension resource still warrants the consolidation agent because that's net-new content the agent needs to integrate, while a deletion is pure pruning.

Error handling is clean: `reset_memory_workspace_baseline(&root).await` is called and on `Err` the run goes to `job::failed(..., "failed_workspace_commit")` with a `tracing::error!` line capturing the underlying error. On success it goes to `job::succeed(..., new_watermark, &raw_memories, "succeeded_empty_input_housekeeping")` which preserves the watermark advance — important so the next phase-2 run doesn't re-process the same housekeeping changes. The new succeed-reason string is greppable and distinct from the regular `succeeded` path which makes telemetry / dashboards able to count savings.

The test update at `startup_tests.rs:221-227` is the right shape: instead of asserting that a phase-2 prompt was sent (which the old behavior did), it now asserts `phase2.requests().len() == 0` after waiting for the file removal and workspace reset. The previous `wait_for_single_request(&phase2)` + prompt-text assertions are deleted entirely — confirming this PR also fixes the Windows-flaky test mentioned in the PR description by removing the model-request expectation that was the source of the flake. Three consecutive `cargo test` runs in CI matches the PR's claim of testing.

One small concern: the predicate at `phase2.rs:158-164` uses `change.path.starts_with(&rollout_summaries_prefix)` and the prefix is built as `format!("{}/", crate::artifacts::ROLLOUT_SUMMARIES_SUBDIR)`. If `ROLLOUT_SUMMARIES_SUBDIR` is ever empty (unlikely) this becomes `/` which matches everything — defensive `debug_assert!(!ROLLOUT_SUMMARIES_SUBDIR.is_empty())` would lock that invariant. Not blocking. PR is in DRAFT so author may polish further before mark-ready.

## Verdict

`merge-after-nits`
