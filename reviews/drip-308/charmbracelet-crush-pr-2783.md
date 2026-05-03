# charmbracelet/crush #2783 — fix: reorder tool results in preparePrompt for strict adjacency providers

- PR: https://github.com/charmbracelet/crush/pull/2783
- Author: somjik-api
- Head SHA: `8985f2f5033fd84837fe668369e465c9e9ad8167`
- Updated: 2026-05-03T00:14:33Z

## Summary
Reworks `internal/agent/agent.go` history assembly so tool result messages are emitted **immediately after** the assistant message that contains their tool calls, satisfying strict-adjacency providers (DeepSeek, etc.) that reject turns where assistant→tool messages aren't directly paired. Replaces `syntheticToolResultsForOrphanedCalls` with `buildToolResultsForCalls` that splices real results inline and only synthesizes for genuinely orphaned calls.

## Observations
- `internal/agent/agent.go` line ~789 introduces `tcToResultMsgs := make(map[string][]message.Message)` — using a slice rather than a single `message.Message` is correct (one tool_call_id can appear in multiple tool messages in pathological histories), but the new code in `buildToolResultsForCalls` (around line ~870) iterates *all* result messages for a given call ID and emits each. If a duplicate result was already going to land later in the original stream order, the dedup is handled via `placedResults[rm.ID] = true`, but this only works because dedup keys on the *message* ID, not the *tool_call_id*. Add a short comment explaining that invariant — it's load-bearing and easy to break.
- Same file ~line 808: the new `if placedResults[m.ID] { continue }` early-skip at the top of the main loop is the right place to handle "already proactively placed" tool messages, but the loop still calls `len(m.Parts) == 0` and `m.ToAIMessage()` checks below the skip. That's fine for correctness but the skip should be the *very first* statement so future readers don't add a side effect above it.
- The synthetic error string at ~line 884 (`"tool call was interrupted and did not produce a result, you may retry this call if the result is still needed"`) is reasonable user-facing copy, but it's hardcoded in Go. If crush localizes operator messages elsewhere, this is a drift point worth flagging.
- No new test in the visible diff for the strict-adjacency reordering — given this is a regression fix targeting DeepSeek behavior, a table test exercising "assistant with N tool_calls + tool messages out-of-order in slice → output is adjacent" would be cheap and would prevent regressions when this code is touched again.
- The deletion of `syntheticToolResultsForOrphanedCalls` (line ~922 in old file, called out as removed in the diff) is correct — its responsibility moves into `buildToolResultsForCalls`. Confirm no other call sites reference the removed function.

## Verdict
`merge-after-nits`
