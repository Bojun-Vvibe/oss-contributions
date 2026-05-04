# openai/codex #20948 — Add session id to Codex runtime analytics events

- SHA: `16d3cb7fbe19e29a0fe3cfe21171db565b8aaa2a`
- State: OPEN, +51/-5 across 7 files

## Summary

Threads a `session_id: String` through six analytics event payloads (ThreadInitialized, Turn, TurnSteer, Compaction, GuardianReview, SubAgent ThreadStarted), persists it on `ThreadMetadataState` so downstream events sourced from cached metadata also emit it, and updates serialization tests for the new field.

## Notes

- `codex-rs/analytics/src/events.rs:107, 380, 429, 459, 516` — `session_id` added as a non-`Option<String>` to every event params struct and serialized first in the JSON object. Consistency is good. Since this is a new wire-level field, downstream consumers (warehouse schemas, dashboards) need a coordinated rollout — empty-string vs `Option<String>` matters here. Recommend `Option<String>` initially with `#[serde(skip_serializing_if = "Option::is_none")]` so deployments where the upstream session id has not yet been plumbed do not emit `"session_id": ""` rows that look like a real value.
- `codex-rs/analytics/src/reducer.rs:148-180, 351-356` — `ThreadMetadataState` now caches `session_id` and the `from_thread_metadata` constructor takes it as the first positional arg. The subagent path at line 351-356 clones `input.session_id` into the cached metadata only when no metadata exists yet (`get_or_insert_with`). If the parent thread already initialized metadata under a *different* session id (re-attached app server), the subagent will emit the parent's session id rather than its own. Worth a sentence in the PR explaining intended semantics: "session_id is per-thread, inherited from initialization." If sessions are app-server-scoped, this is correct; if they're per-attach, this drops information.
- `codex-rs/analytics/src/reducer.rs:370-380` — switching `thread_connection_or_warn` to `thread_context_or_warn` (returning a 2-tuple of connection + metadata) is the right refactor — guardian events now read `session_id` from metadata rather than re-passing it through every call site. Worth verifying every other caller of the old single-return helper still compiles after the rename; a sibling helper is expected.
- `codex-rs/core/src/agent/control.rs`, `codex_delegate.rs`, `session/mod.rs` — only +1 line each; these are call-site updates that pass `session_id` into the analytics ingestion. Clean blast radius.
- Validation note in PR body: `cargo test -p codex-analytics -p codex-core responses_headers --no-default-features` blocked by missing host `libcap`/`pkg-config`. The `codex-analytics` package tests pass; `codex-core` integration tests for the new wiring are not exercised in CI for this PR. A follow-up CI fix or a feature-gated stub would close the gap.

## Verdict

`merge-after-nits` — wiring is correct and low-risk inside the analytics crate. Reconsider `Option<String>` for the wire field to avoid emitting empty session ids during partial rollouts, and document the per-thread-vs-per-attach semantics for the cached metadata path. Resolve the `codex-core` test gap or note it as a known follow-up before merging.
