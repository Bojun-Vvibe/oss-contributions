# PR #19855 — Continue sampling after assistant chunks

- **Repo**: openai/codex
- **PR**: #19855
- **Head SHA**: `fa64a935`
- **Author**: surajs-oai
- **Size**: +10 / -1 across 1 file
- **URL**: https://github.com/openai/codex/pull/19855
- **Verdict**: **merge-as-is**

## Summary

Ten-line surgical fix to the sampling loop's "should we send another
provider request" decision in `core/src/session/turn.rs:try_run_sampling_request`.
Before: a turn ended (`needs_follow_up = false`) whenever the response
ended with no tool call to dispatch. After: a turn additionally
continues sampling when the last item was either a `Reasoning` block
*or* an `assistant` message with `phase: Some(MessagePhase::Commentary)`.
This is the corresponding follow-up to #19832 (drip-117) which added
the `phase` round-trip on `ResponseInputItem::Message`: now that
commentary messages are recognized as not-yet-final, the sampling loop
correctly keeps the turn alive after one is emitted, instead of
treating the commentary as the terminal assistant utterance.

## Specific changes

- `codex-rs/core/src/session/turn.rs:1902` — declare
  `let mut should_continue_sampling_assistant = false;` next to
  `needs_follow_up`.
- `codex-rs/core/src/session/turn.rs:2009-2014` — compute the per-item
  predicate at the same point `output_result.needs_follow_up` is
  evaluated:
  ```rust
  let output_item_should_continue_sampling_assistant = match &item {
      ResponseItem::Reasoning { .. } => true,
      ResponseItem::Message { role, phase, .. } => {
          role == "assistant" && matches!(phase, Some(MessagePhase::Commentary))
      }
      _ => false,
  };
  ```
- `codex-rs/core/src/session/turn.rs:2032` — `should_continue_sampling_assistant = output_item_should_continue_sampling_assistant;`
  *assigns* (not OR-equals) per item — this is correct because only the
  *last* item's value should drive the decision (a mid-stream reasoning
  block followed by a final tool call should follow the tool-call's
  needs_follow_up, not the reasoning's `true`).
- `codex-rs/core/src/session/turn.rs:2157` — break value changes from
  `needs_follow_up` to `needs_follow_up || should_continue_sampling_assistant`.
  Note: this OR-combine *is* correct here because the break point is
  outside the per-item loop; at that point we want either condition to
  trigger another sampling round.

## Risks

- **No new test in this diff**: the surrounding loop is end-to-end
  tested by integration suites (the surrounding code at `:1899-2145`
  is the orchestrator for a sampling round, exercised on every turn),
  but a unit-level pin that constructs a `ResponseItem::Message`
  with `phase: Some(MessagePhase::Commentary)` and asserts
  `should_continue_sampling_assistant` is true at break would protect
  against the next refactor silently reverting the case match.
- **Per-item assignment vs. OR-equals**: subtle but correct. If the
  last item is a `Function` call, `output_item_should_continue_sampling_assistant`
  is false, which overwrites a prior `true` from an earlier reasoning
  block — that's the right behavior because the function call carries
  its own `needs_follow_up` and the reasoning is no longer the
  trailing item. The implicit invariant is "only the trailing item's
  shape determines whether the assistant is still talking", which the
  assignment-not-OR encoding gets right.
- **Phase enum is an `Option`**: `phase: Some(MessagePhase::Commentary)`
  matches only when phase is *explicitly* commentary. A `phase: None`
  message — which is what every legacy persisted assistant message has
  before #19832 lands and propagates through rollouts — does not
  trigger continuation, which is the correct backward-compatible
  default (legacy messages were always treated as terminal).

## Verdict

**merge-as-is**: ten lines, one decision point, correct pattern-match
shape, correct per-item assignment vs. OR-equals semantics, paired
with the upstream phase round-trip from #19832. Worth landing without
gating on a unit test because the change is small and the integration
coverage is already there — but a follow-up PR adding a unit-level
pin on the `MessagePhase::Commentary` branch would make the
regression cost smaller.

## What I learned

Two-PR change-pairs like this — #19832 adds the `phase` field on the
wire and threads it through the conversion layer; #19855 (this) adds
the consumer that *uses* the field — are easier to review when each
PR is small and the upstream PR's reviewer has already pinned the
"phase round-trips" behavior with a regression test. The downstream
consumer can then be a 10-line drop-in. The cost model: if you bundle
both into one PR, the reviewer has to hold the entire wire-format +
state-machine shape in their head at once. Splitting them lets each
PR's reviewer focus on one invariant. The risk to manage is the
intermediate state — if #19832 lands but #19855 doesn't, commentary
messages round-trip on the wire correctly but the assistant
prematurely stops talking. The two-PR shape is acceptable when the
intermediate state is *strictly less broken* than the previous state
(here: a phase that's preserved-but-ignored is no worse than no
phase at all), which is the case here.
