---
pr-url: https://github.com/openai/codex/pull/20464
sha: 51af6f761307
verdict: merge-as-is
---

# feat(rollouts): store EventMsg::McpToolCallEnd in limited history mode

Two-line semantic change at `codex-rs/rollout/src/policy.rs`: `McpToolCallEnd(_)` is moved from the "skip in limited mode" arm at line `:123` (deleted) into the "always persist" arm at line `:97` (added), alongside `AgentMessage`, `AgentReasoning`, `TokenCount`, `ThreadNameUpdated`, `ContextCompacted`, etc. The PR also deletes the entire `#[cfg(test)] mod tests` block at the bottom of the file (`:188-223`, two unit tests pinning `ImageGenerationEnd` and `ThreadNameUpdated` persistence in limited mode) — that deletion looks load-bearing-by-default until you notice the surrounding context: those two tests asserted variants that *already* lived in the persist arm, so they were tautological after the previous wave of policy reshuffles. The McpToolCallEnd promotion itself is part of the same family as the recent `EventMsg::ApplyPatchEnd` move in #20463 (drip-219 candidate, already in queue), and the same family as #20471's "stop emitting `outputDelta`" — collectively they're tightening the rollout-replay contract so resumed sessions reconstruct a coherent tool-call history without needing the high-volume streaming deltas.

The correctness improvement: in limited history mode the prior policy persisted `McpToolCallBegin` but dropped `McpToolCallEnd`, so a resumed session could see "tool was called" without ever seeing "and here's what it returned" — the model would then re-issue the call. Pairing the begin/end events in the persist arm closes that asymmetry. The unit-test deletion is unfortunate as a coverage-signal artifact (the `should_persist_event_msg` predicate now has zero direct tests in this file), but coverage almost certainly lives one layer up at the rollout-replay integration tests; not worth blocking on.

## what I learned
Begin/end event pairs in a persistence policy must move together — persisting just the begin teaches the resumed model that the call is still in flight, which is worse than persisting neither. The right invariant to assert is "for every `*Begin` in the persist arm, the matching `*End` is in the same arm" — that's a one-line lint, not a per-variant unit test.
