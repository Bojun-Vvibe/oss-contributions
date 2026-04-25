---
pr: 19481
repo: openai/codex
sha: 7eae03d21fa4d818addfa637696d48173550f8d5
verdict: merge-after-nits
date: 2026-04-25
---

# openai/codex#19481 — Remove ghost snapshots

- **URL**: https://github.com/openai/codex/pull/19481
- **Author**: pakrym-oai

## Summary

Retires the `ghost_snapshot` / `GhostCommit` family from the live
Responses API surface and the generated SDK/schema artifacts, while
keeping the legacy `undo` feature key parseable so existing user
configs don't fail to load. The undo task itself becomes a stub that
emits `UndoCompletedEvent { success: false, message: "Undo is no
longer available." }`. Cleanup spans protocol JSON schemas, TS
definitions, core history/compaction/rollout/tasks, telemetry timing,
and tests.

## Reviewable points

- **Public protocol**: `app-server-protocol/schema/json/{ClientRequest,
  v2/RawResponseItemCompletedNotification, v2/ThreadResumeParams,
  …}.json` all drop the `GhostCommit` definition and the
  `ghost_snapshot` ResponseItem variant. `schema/typescript/
  GhostCommit.ts` is deleted; `ResponseItem.ts` removes the
  `{ "type": "ghost_snapshot", ghost_commit: GhostCommit }` arm. This
  is a **breaking change** for any out-of-tree consumer that pattern-
  matches on that variant. Worth a CHANGELOG/MIGRATION note calling
  out that downstream SDK consumers must drop the variant from their
  unions.

- **Feature flag handling**: `features/src/lib.rs:71-75` retains the
  `Feature::GhostCommit` enum variant but flips the `FeatureSpec` for
  key `"undo"` from `Stage::Stable` to `Stage::Removed` (line 620).
  The match arm in `Features::from_sources` adds an explicit `"undo"
  => continue;` (line 392) so a stray `undo = true` in a user
  features.toml is silently ignored rather than rejected. New tests
  `undo_is_removed_and_disabled_by_default` and
  `from_sources_ignores_removed_undo_feature_key` (in features/src/
  tests.rs) cover this. Pattern matches the prior
  `image_detail_original` removal — consistent.

- **Undo task stub**: `core/src/tasks/undo.rs` (around line 60 of
  the changed file) drops everything that read history, found the
  most-recent `ResponseItem::GhostSnapshot`, and called
  `restore_ghost_commit_with_options` from `codex-git-utils`. The
  replacement is just emitting `UndoCompletedEvent { success: false,
  message: Some("Undo is no longer available.".to_string()) }`. The
  message is fine but doesn't mention the feature is fully removed
  vs. temporarily disabled — consider `"Undo has been removed in this
  release; see CHANGELOG."` so support tickets are easier to triage.

- **TTFT telemetry**: `core/src/turn_timing.rs:182` removes the
  `ResponseItem::GhostSnapshot` arm from
  `response_item_records_turn_ttft`. Correct — that variant no longer
  exists. Worth confirming the historical rollout fixtures (which may
  contain old `ghost_snapshot` items from before this PR landed) still
  parse via the `core/src/context_manager/history.rs` changes; if a
  user resumes a thread that contains an old ghost-snapshot item, the
  loader should now treat it as `ResponseItem::Other` rather than
  failing to deserialize. A targeted resume-from-old-rollout test
  would close that loop.

- **`codex-git-utils` shrinkage**: Cargo.toml/README/lib.rs in the
  git-utils crate also shrink. Confirm that no external crate (in
  `codex-rs/Cargo.lock`) still pulls the old API surface — the lockfile
  diff (`Cargo.lock 0+/1-`) suggests a single dep removed but no other
  reverse callers, which is the right shape.

- **Schema diff size**: 5 separate `*.schemas.json` files all see the
  same ~52-line removal, with parallel TS file updates. Mechanical and
  consistent — good. No risk of partial removal.

## Rationale

This is a clean retirement: protocol and code surfaces are removed
together, the legacy `undo` config key is silently tolerated, and
tests cover both the disabled-default and ignored-source paths. The
two nits — (a) sharper user-facing "removed" message, and (b) an
explicit resume-from-old-rollout regression test — are non-blocking
but would tighten the deprecation story.

## What I learned

When removing a feature that wrote items into a persisted protocol
stream, the test that catches the most regressions isn't "the new
empty undo path emits the right message" — it's "an old persisted
stream containing the now-removed item still deserializes". This PR
mostly does that via `ResponseItem::Other` fallthrough, but the
explicit regression test is what makes the rollback story credible.
