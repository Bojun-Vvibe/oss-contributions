# openai/codex #20080 — [codex-analytics] prevent stale guardian events from satisfying reused reviews

- PR: https://github.com/openai/codex/pull/20080
- Head SHA: `d1d02d522a3f244faa61472e4b46b53e20b6b4a2`
- Files: `codex-rs/core/src/guardian/review_session.rs`

## Citations

- `review_session.rs:702` — `submit_result` destructure changes from `Ok(Ok(_)) => {}` to `Ok(Ok(child_turn_id)) => child_turn_id`, capturing the per-turn id returned by `Op::ReviewSubmit` for later filter use.
- `review_session.rs:716` — `wait_for_guardian_review` now takes `expected_turn_id: &str` (was untyped before).
- `review_session.rs:799` — the central event-pump match arm `Ok(event) if !event_matches_turn(&event, expected_turn_id) => {}` discards events whose `event.id` or per-msg `turn_id` don't match.
- `review_session.rs:842-855` — new `event_matches_turn(event, expected_turn_id)` checks `event.id == expected_turn_id` first, then for `TurnComplete`/`TurnAborted` cross-checks the message-level `turn_id` field too (defense-in-depth: belt + suspenders against an `Event` whose `id` and `msg.turn_id` disagree under reuse).
- `review_session.rs:962-985` — `interrupt_and_drain_turn` re-signed to take `expected_turn_id`, and the drain loop now requires *both* `event_matches_turn` AND `EventMsg::TurnAborted | TurnComplete` before exiting; this prevents the drain from latching onto a stale `TurnComplete` for a previously-reused review.
- `review_session.rs:990+` — new test scaffolding (`test_review_session`, `turn_complete_event`, `turn_aborted_event`) builds the channels needed to inject a stale event into a reused session and assert it is filtered.

## Verdict

`merge-as-is`

## Reasoning

This is a model-class fix for a class of analytics corruption that's notoriously hard to reproduce: events that are *delivered* but bound to a logically-prior turn-id, observed only because the underlying review-session is being reused. The PR description's framing — "non-null TTFT with sub-10 ms completion latency and zero token deltas on `trunk_reused` reviews" — is the smoking gun: those numbers are physically impossible if TTFT and completion latency are both attributed to the same turn, and the only way they can co-occur is if TTFT is sourced from event A (the *new* turn's first-token) while completion latency is sourced from event B (the *previous* turn's already-queued `TurnComplete`). The fix removes that cross-attribution by rejecting any event whose `id` doesn't match the captured `child_turn_id`.

Three things make the implementation right rather than just plausible:

1. **The captured turn-id is the source of truth** — it comes back from `Op::ReviewSubmit`'s response in the same protocol round-trip that establishes the new turn, so by construction it's correctly paired with the new review. No clock-based heuristic, no "drop everything older than X ms" — the matching is purely structural on a string id.

2. **`event_matches_turn` cross-checks the per-message `turn_id` for terminal events** (`TurnComplete.turn_id`, `TurnAborted.turn_id`) in addition to the envelope's `event.id`. For non-terminal events it falls through to `_ => true` after the envelope check, which is the right asymmetry: terminal events are what the analytics depends on, so they get the strictest filter; intermediate events (`AgentMessage`, etc.) are accepted on envelope match alone since their message variants don't carry a turn_id field.

3. **`interrupt_and_drain_turn` is updated symmetrically.** This is the easy thing to miss in a fix like this — the timeout/cancel paths drain the channel by waiting for a `TurnAborted | TurnComplete`, and if you don't filter the drain by turn-id too, you could exit the drain on a stale event, leaving the *real* abort/complete still queued for the next reused review to mis-attribute. The PR catches this and applies the same `event_matches_turn` predicate (line 967) before the matches!() check.

The new test infrastructure (`test_review_session` builds a fake `GuardianReviewSession` over `async_channel::bounded(4)`/`unbounded`, with helper constructors `turn_complete_event` and `turn_aborted_event` that mint events with deliberately mismatched `id` / `turn_id` shapes) is exactly what's needed to pin this behavior; these tests will catch any future refactor that strays from the structural-id-match invariant.

Risk surface is small. The only behavioral change for non-reused trunks is "an event with `event.id != expected_turn_id` is silently dropped" — but in the non-reused case there's no producer for such an event, so the `_ => {}` arm is effectively dead code outside of the targeted bug. The change is an analytics correctness fix with no user-visible behavior change beyond the impossible-data-points going away.

Ship.
