# openai/codex#20515 — [codex-analytics] expose item timing through app server

- **PR**: https://github.com/openai/codex/pull/20515
- **Head SHA**: `158fe58e7c5ed957613ab91e99067aeaa98d3de9`
- **Files**: 28 files; protocol JSON schemas + TypeScript fixtures + `event_mapping.rs`/`item_builders.rs`/`thread_history.rs`/`v2.rs` plus app-server tests. Stack PR (depends on #20514).
- **Verdict**: **merge-after-nits**

## Context

Second layer of the codex-analytics stack: surface per-item lifecycle timing (`startedAtMs`, `completedAtMs`, `durationMs`) on the *app-server public* protocol once #20514 has landed core production of those fields. Stack ordering is correct — protocol producer (#20514) before public wire/history layer (#20515) — so generated-schema churn is reviewable independently from core logic. Touches every item subtype (CommandExecution, PatchApply, McpToolCall, DynamicTool, CollabTool, WebSearch, ImageGeneration) plus their wire shapes in `ServerNotification.json`, `ItemStartedNotification.json`, `ItemCompletedNotification.json`, `ThreadResumeResponse.json`, etc.

## What's right

- **Generated schema is symmetric across all item types.** `ServerNotification.json:3317-3398` (Command), `:3402-3460` (PatchApply), `:3452-3508` (McpToolCall), `:3536-3592` (DynamicTool), `:3617-3690` (CollabTool), `:3729-3760` (WebSearch), `:3801-3848` (ImageGeneration) — every variant gets the same `startedAtMs`/`completedAtMs`/`durationMs` triple with consistent description ("Unix timestamp (in milliseconds) when X execution started/completed, if known."). Symmetric naming is exactly right for a public wire format reviewers will need to consume.
- **`int64` + nullable shape preserves back-compat.** Every new field is `"type": ["integer", "null"], "format": "int64"` so older app-server clients that don't know about `startedAtMs` continue to deserialize fine. Generated TypeScript at `app-server-protocol/schema/typescript/v2/ThreadItem.ts` (+74/-2) follows the same shape.
- **`thread_history.rs` reconstruction preserves timing.** The +90 lines in `app-server-protocol/src/protocol/thread_history.rs` are the load-bearing piece — when a client resumes a thread, the reconstructed item events must carry the same timing they had at original-emission, not be stripped to `None`. PR body's "preserves timing through thread-history reconstruction and item builders" claim is the explicit risk that needs this code path.
- **Test coverage spans dynamic_tools, mcp_tool, thread_resume, and turn_start.** `tests/suite/v2/dynamic_tools.rs` (+8), `mcp_tool.rs` (+2), `thread_resume.rs` (+21/-7), `turn_start.rs` (+43/-14) — covers every subtype the schema added timing to. Per PR body verification: `cargo test -p codex-app-server-protocol`.
- **`bespoke_event_handling.rs` (+11)** wires the producer-side event mapping for the variants that need bespoke treatment beyond the generic `event_mapping.rs:550+` flow.
- **Stack tag explicit in PR body.** `* #18748 / * #18747 / * #17090 / * #17089 / * #20239 / * __->__ #20515 / * #20514 / * #20300` — reviewers know the dependency order and can verify each PR in turn.

## Risks / nits

- **Generated-schema diff is 21 of 28 files (75% of the diff).** The actual hand-written changes are concentrated in `event_mapping.rs` (+50), `item_builders.rs` (+19), `thread_history.rs` (+90), `v2.rs` (+63), `bespoke_event_handling.rs` (+11). Recommend PR body explicitly call out "generated files: schema/json/v2/*.json, schema/typescript/v2/ThreadItem.ts; hand-written: src/protocol/*.rs, app-server/src/bespoke_event_handling.rs" so reviewers know to skip the schema files (after one spot-check confirms regeneration ran cleanly) and focus on the hand-written code.
- **No assertion that `completedAtMs >= startedAtMs` at the wire layer.** A clock-skew bug in core production (which #20514 is responsible for) could surface as `completedAtMs < startedAtMs`, which is meaningless to consumers. Recommend either a debug-assert in `event_mapping.rs` or a doc comment explicitly noting that consumers should *not* trust the ordering invariant and should clamp `durationMs = max(0, completedAtMs - startedAtMs)` if computed client-side.
- **`durationMs` is also stored as a separate field on most variants, not just derived.** This is intentional (matches PR body: "Repeating `started_at_ms` on end events lets reducers consume a complete interval from the completion event itself") but creates a redundancy-divergence risk: if the producer emits `startedAtMs=100, completedAtMs=200, durationMs=99`, which one is authoritative? Recommend a doc comment in the protocol module declaring the source-of-truth precedence (e.g., "if `durationMs` is present, prefer it; otherwise compute from `completedAtMs - startedAtMs`").
- **`item_builders.rs:+19`** changes need a quick check that the builder defaults for the new fields are `None` (not `Some(0)` or current-clock-time), so a producer that hasn't been updated to set them yet doesn't accidentally synthesize misleading zero values. Quick spot-check of the diff suggests this is correct, but worth pinning in a test.
- **Stack PR ordering: this PR cannot land before #20514.** Worth flagging in PR body or as a CI/merge-queue dependency so a maintainer doesn't accidentally pre-merge this and break the producer-side core compile.

## Verdict

**merge-after-nits.** Mechanically-applied symmetric schema expansion across every item subtype with reconstruction support and per-subtype test coverage. The "repeat `started_at_ms` on end events for self-contained intervals" discipline is a deliberate decision that the PR body justifies well. Strongest nits: explicit generated-vs-hand-written file callout in PR body, source-of-truth precedence doc for the redundant `startedAtMs/completedAtMs/durationMs` triple, and a test pinning that builders default the new fields to `None` not synthesized values.
