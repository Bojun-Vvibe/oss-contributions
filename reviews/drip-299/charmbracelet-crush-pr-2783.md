# charmbracelet/crush PR #2783 — fix: reorder tool results in preparePrompt for strict adjacency providers

- URL: https://github.com/charmbracelet/crush/pull/2783
- Head SHA: `8985f2f5033fd84837fe668369e465c9e9ad8167`
- Author: somjik-api (Somjik)

## Summary

In `internal/agent/agent.go`, replaces the existing
`syntheticToolResultsForOrphanedCalls` path with a new
`buildToolResultsForCalls` that:

1. Pre-builds a `tcToResultMsgs map[string][]message.Message` of every
   tool result keyed by its tool-call ID.
2. When iterating `msgs`, after emitting an assistant message, **eagerly
   places the matching tool-result messages directly after it** and
   marks them `placedResults[m.ID] = true` so they're skipped when later
   encountered in the natural order.
3. Falls back to the same synthetic-error tool-result payload for tool
   calls that have no result anywhere in history.

Adds `TestPreparePrompt_NonAdjacentToolResults` (~190 lines visible).

## Review

Solid fix for a real provider-strictness issue: DeepSeek (and to some
extent strict Anthropic adjacency rules) reject a turn when an assistant
message containing `tool_use` is followed by anything other than a `tool`
message. The DB ordering can interleave background events between them,
which historically locked the conversation.

Specific points:

- `agent.go:786-822` — building `tcToResultMsgs` and `placedResults` in
  the same pre-pass over `msgs` is O(n) and avoids a second walk. Good.
- `agent.go:~860` — `buildToolResultsForCalls` iterates the assistant's
  `ToolCalls()` in declaration order and appends the matched real
  results before the synthetic-error block. Confirm that when there are
  multiple tool results for the same `ToolCallID` (shouldn't happen, but
  the type allows it), the loop emits both — currently it does, which
  may double up on the wire. A `len(resultMsgs) > 1` warn-log would help
  track this if it ever happens in prod.
- The synthetic-error message keeps the same English text, which is
  important: existing telemetry / log scrapers keying on it continue to
  match.
- `filterOrphanedToolResults` is still called on the natural-order pass
  for any tool message not already placed. The `placedResults[m.ID]`
  early-`continue` at the top of the outer loop is the correct guard.

Risk: if the same tool message ID appears in `tcToResultMsgs` for
*multiple* assistant tool calls (shouldn't, but a corrupted history
could), it gets emitted only once and the second assistant's call
becomes orphan-synthesized. That's the safe failure mode, but worth
calling out in a comment.

## Verdict

`merge-after-nits` — logic is right; add a one-line comment explaining
the `placedResults[rm.ID]` interaction with multi-call assistant
messages, and consider warn-logging the duplicate-results case.
