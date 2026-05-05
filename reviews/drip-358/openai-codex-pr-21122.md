# openai/codex PR #21122 — Add turn_id to Codex skill invocation analytics

- Repo: `openai/codex`
- PR: #21122
- Head SHA: `6059e18aa617`
- Author: `edwardysun3`
- Scope: 3 files, +3 / -0
- Verdict: **merge-as-is**

## What it does

Adds a `turn_id: Option<String>` field to the `SkillInvocationEventParams` struct so that skill-invocation analytics events carry both the `thread_id` they already had and the `turn_id` they were missing.

Three additive changes:

- `codex-rs/analytics/src/events.rs:85` — adds `pub(crate) turn_id: Option<String>,` to `SkillInvocationEventParams`.
- `codex-rs/analytics/src/reducer.rs:498-499` — populates `turn_id: Some(tracking.turn_id.clone())` next to the existing `thread_id` when the reducer materializes the event from `tracking`.
- `codex-rs/analytics/src/analytics_client_tests.rs:1838` — extends the existing reducer test fixture to assert the new field on the emitted JSON.

## Why this is fine

- **Pure additive.** New optional field on an internal-crate analytics struct, populated from a value (`tracking.turn_id`) that is already in scope at the only callsite. No protocol break, no schema migration, no behavioural change to skill execution itself.
- **Symmetric with `thread_id`.** The reducer already had `thread_id: Some(tracking.thread_id.clone())`; pairing it with `turn_id` is the obviously-correct shape — analytics on skill invocations is much less useful if you can't bucket multiple invocations within the same model turn.
- **Test coverage updated in the same PR.** The `reducer_ingests_skill_invoked_fact` test (`analytics_client_tests.rs:1825-1860` region) was updated to include `"turn_id": "turn-1"` in its expected JSON, so a future refactor that drops the field will fail loudly.
- **No downstream readers to break.** This is the producer side; consumers that don't know the field will just ignore it (standard JSON-payload analytics shape).

## Nits (optional, non-blocking)

- The struct now has `Option<String>` for both `thread_id` and `turn_id`. In practice the reducer always emits `Some(...)` for both; if neither is ever legitimately `None`, a follow-up could tighten to `String` to make that invariant compile-checked. Out of scope for this PR.

## Verdict
**merge-as-is** — minimal, additive, test-covered, single-purpose. Nothing to argue about.
