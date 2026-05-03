# openai/codex PR #20915 — frodex: pin rollout references by segment

- Link: https://github.com/openai/codex/pull/20915
- SHA: `567b66a6e9f39674f3c94c13ed6526f2e6edade1`
- Author: friel-openai
- Stats: +4352 / −1901, ~30+ files (large schema regen + protocol/core changes)

## Summary

Introduces a separate `SegmentId` for rollout-storage references while keeping `ThreadId` as the runtime session identity. `ForkReference` is changed from a single `thread_id` to `{ thread_id, segment_id }` so that a fork reference remains stable when the session later writes another rollout JSONL segment, or when an old segment is archived/rotated. Also keeps the recent compaction cleanup (drop duplicate contiguous user messages, filter unified-exec max-process warnings by prefix instead of relying on the deprecated `exclude_from_compaction` field).

## Specific references

- `codex-rs/app-server-protocol/schema/json/ServerNotification.json` L897–L998: regen adds new `ContentItem`, `FunctionCallOutputBody`, and `FunctionCallOutputContentItem` schema variants. The `FunctionCallOutputContentItem` doc (L1391-area) says "subset of ContentItem with the types we support as function call outputs" — good defensive framing, but the actual subset isn't enforced anywhere in the schema (it's a separately-defined `oneOf`). If those two ever drift, only review will catch it. A short serde test asserting `FunctionCallOutputContentItem` ⊆ `ContentItem` for the supported variants would be cheap insurance.
- The schema diff itself is +882/-27 in `ServerNotification.json` alone — almost all auto-generated. Reviewers should be looking only at the matching Rust source-of-truth changes (likely under `codex-rs/app-server-protocol/src/`). Worth mentioning in the PR description that the JSON schema files are derived, so reviewers don't waste cycles on them.
- PR body explicitly calls out that fork-only release workflow and default-on Frodex feature flags are *not* in this stack — that is the correct scoping. This PR should land behind the existing Frodex flag and not flip any defaults.
- The `SegmentId`/`ThreadId` split is the right model: `ThreadId` stays the user-visible identity and continues to flow into `app-server` ops, TUI routing, state DB rows, and model headers; `SegmentId` only resolves a specific JSONL on disk. This means existing consumers that key off `thread_id` keep working, and only the rollout subsystem learns the new id.
- Validation listed: `cargo test -p codex-rollout`, `cargo test -p codex-core thread_rollout_truncation -- --nocapture`, `cargo test -p codex-app-server-protocol`, `cargo build -p codex-cli --bin codex` — covers the three crates that own the new types. Missing from the list: any TUI-side test that exercises a fork after a segment rotation, which is the exact scenario this PR is supposed to fix. A small integration test that (1) creates a thread, (2) rolls a new segment, (3) forks and checks the reference still resolves, would lock the regression in.
- Stack base is `dev/friel/frodex-129-fork-command-debug` — PR shouldn't land before that base is in.

## Verdict

verdict: needs-discussion

## Reasoning

The model is right and the scoping (no flag flip, no fork-only release workflow) is conservative. The discussion points before merging: (1) confirm reviewers only need to look at Rust sources, not the +880-line generated JSON schema; (2) decide whether a `SegmentId`-rotation integration test belongs in this PR or the next stack entry; (3) confirm migration story for any persisted `ForkReference`s currently on disk that only have `thread_id` (the diff is silent on backward-compat decode). Once those are cleared this is a `merge-after-nits`. Verdict is needs-discussion because the migration-compat question is a release-blocker if the answer is "no fallback decode".
