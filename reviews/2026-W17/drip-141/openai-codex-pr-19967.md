# openai/codex#19967 — Stabilize memory Phase 2 input ordering

- **PR**: https://github.com/openai/codex/pull/19967
- **Author**: @jif-oai
- **Head SHA**: `6bc764181a46f266a98c149393d68d540c131026`
- **Base**: `main`
- **State**: MERGED
- **Scope**: +85 / -56 across 5 files

## Summary

Phase 2 of codex's memory consolidation pipeline reads top-N stage-1 outputs from SQLite ranked by usage, then writes them into `raw_memories.md` and `rollout_summaries/` on disk. Workspace-diff reasoning runs against this on-disk state. The bug: the SQL `ORDER BY usage_count DESC, last_usage DESC, ...` produced a recency-ranked list that was *also* the on-disk order. Every Phase 2 run that bumped a `usage_count` reordered the file (and thus made workspace-diff show false moves), and the LLM consolidator was being told to read it "latest first" as a recency heuristic — coupling the ranking signal to the disk layout. Fix decouples them: SQL still ranks by usage to *select* the top-N rows, but the outer SELECT re-sorts those rows by `thread_id ASC` before the consumer sees them. `raw_memories.md` is now stable across runs except when the *set* of selected rows changes, which is exactly the signal workspace-diff is supposed to surface.

## Diff anchors

- `codex-rs/state/src/runtime/memories.rs:355-394` — SQL refactor wraps the original ranked SELECT in a derived table `selected` and adds a final `ORDER BY selected.thread_id ASC`. This is the right shape because it preserves the existing top-N selection semantics (`LIMIT ?` stays inside the inner query, ranked by `usage_count DESC, ...`) while changing only the on-disk order of those selected rows. A naive "just change the outer ORDER BY" would have changed which rows are *selected* whenever ties exist, silently reshaping the corpus.
- `codex-rs/state/src/runtime/memories.rs:335-340` — doc comment is updated honestly: "eligible rows are *ranked* by usage_count DESC ..." (was "ordered"), and a new bullet "the selected top-N rows are returned in stable `thread_id ASC` order" is added. The word change `ordered → ranked` matters — it's the conceptual boundary that the SQL refactor enforces.
- `codex-rs/memories/write/src/storage.rs:57` — `body.push_str("Merged stage-1 raw memories (latest first):\n\n")` becomes `("(stable ascending thread-id order):\n\n")`. The user-visible string in the rendered file matches the new invariant. Without this, downstream LLM reads of `raw_memories.md` would still see "latest first" framing and prefer top entries as recency.
- `codex-rs/memories/write/templates/memories/consolidation.md:120-130` — the consolidator prompt is rewritten to remove every "ordered latest-first" / "recency ordering as a major heuristic" / "newest portion first" phrase. The replacements name the new sources of recency (`updated_at`, workspace diff, rollout content). This is load-bearing — leaving the old phrasing in the prompt would re-introduce the coupling at the LLM layer even after the file ordering was fixed.
- `codex-rs/memories/README.md:94-133` — docs updated symmetrically: `raw_memories.md` description changes from "merged raw memories, latest first" to "merged raw memories, stable ascending thread-id order"; the Phase 2 selection paragraph at `:127-133` rewrites the pipeline description to name the explicit re-render-in-thread-id-order step. Three-surface symmetry (SQL → file body → docs → prompt) is the correct shape for a coupling-removal change.
- `codex-rs/state/src/runtime/memories.rs:1274-1278, 2849-2851, 3265-3268, 3795-3798` — test refactor introduces `fn stable_thread_id(value: &str) -> ThreadId` and replaces `Uuid::new_v4()` with deterministic UUID literals (`...000001`, `...000002`, ...) at four test sites. Without this, the tests would still pass under the old ordering by accident (random UUIDs sort randomly) but would flake under the new ordering. The `vec![thread_id_b, thread_id_c]` assertion at `:2944` and `vec![thread_id_c, thread_id_d]` at `:3395` pin the new contract: top-N selected by usage rank, then ordered by ascending thread ID.
- `codex-rs/state/src/runtime/memories.rs:3870` — secondary test edit changes `get_phase2_input_selection(/*n*/ 3, ...)` to `(/*n*/ 1, ...)`. This narrows the assertion surface but I'd want to confirm it isn't a coverage regression (see nits).
- `codex-rs/core/src/session/multi_agents.rs:24-25` — drive-by addition of `SessionSource::Internal(_)` to the `None` arm of `usage_hint_text`. Symmetric with the existing `SessionSource::SubAgent(_) => None` and presumably matches a recent enum widening; harmless but it's drive-by relative to the PR title and should be called out in the body.

## What I'd push back on (nits)

1. **The `n=3 → n=1` change at `:3870` looks like a coverage regression**. The original test exercised the top-3 case; under the new ordering the top-3 of 3 should be `[a, b, c]` in ascending order regardless. Reverting to `n=3` and updating the expected vector to `[thread_a, thread_b, thread_c]` would preserve coverage of the multi-row stable-order path and not just the single-row path.
2. **`stable_thread_id` is duplicated as a free function inside the `mod tests` block**. Three test functions reach for it; consolidating into a single helper is what the PR does. Good. But the helper string format `"00000000-0000-4000-8000-00000000000N"` is repeated at four sites — a 6-line `fn n(i: u8) -> ThreadId { stable_thread_id(&format!("00000000-0000-4000-8000-{:012}", i)) }` would tighten that further.
3. **`SessionSource::Internal(_)` drive-by at `core/src/session/multi_agents.rs`** belongs in a separate commit or at least a PR-body callout. It's correct (the new variant should not contribute usage-hint text any more than `SubAgent` does), but mixing it into the memory-ordering PR makes bisects noisier.
4. **No test for the workspace-diff invariant directly**. The PR's stated motivation is "stop workspace-diff from showing false moves." The included tests pin the *ordering* invariant (which is necessary), but a test that runs Phase 2 twice with the same input set and asserts `git diff` is empty (or that the file's bytes are identical) would pin the *user-visible* invariant the PR is about. This is a follow-up nit, not a merge blocker.

## Verdict

**merge-after-nits** — the SQL derived-table refactor is the right shape (preserves selection semantics, changes only post-selection order), the four-surface symmetry (SQL/file body/docs/prompt) is unusually thorough, and the deterministic-UUID test refactor proves the author understood that the old random-UUID tests were passing by luck. Recommend (1) revert the `n=3 → n=1` narrowing, (2) move the `multi_agents.rs` change to a separate commit, before merge — both 5-minute fixes.

Repo coverage: openai/codex (memory consolidation runtime + prompt + docs).
