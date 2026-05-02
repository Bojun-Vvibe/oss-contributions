# charmbracelet/crush PR #2783 — fix: reorder tool results in preparePrompt for strict adjacency providers

- Head SHA: `8985f2f5033fd84837fe668369e465c9e9ad8167`
- URL: https://github.com/charmbracelet/crush/pull/2783
- Size: +234 / -35, 2 files
- Verdict: **merge-after-nits**

## What changes

`internal/agent/agent.go` rewrites the assembly path in `preparePrompt`
that maps DB-ordered messages into the API-ready history. Previously the
loop emitted messages in DB order and only injected synthetic results
for fully orphaned tool calls. Strict-adjacency providers (DeepSeek,
some Bedrock variants) reject turns where a `tool_use` block is
followed by anything other than a matching `tool_result`. With
sub-agents and concurrent writers, real tool results can land *after*
an interleaved user message in the DB, producing a valid log but an
invalid API request.

New flow (lines 786–838):
1. First pass builds `tcToResultMsgs map[string][]message.Message`
   mapping each tool_call_id to the message(s) that contain its
   result.
2. Second pass tracks `placedResults map[string]bool` and skips
   messages whose IDs were already proactively placed.
3. After emitting each assistant message,
   `buildToolResultsForCalls(m, tcToResultMsgs, placedResults)`
   (lines 850–890) appends real results when known and a synthetic
   error tool-result otherwise.
4. The old `syntheticToolResultsForOrphanedCalls` is removed.

A new test `TestPreparePrompt_NonAdjacentToolResults` (agent_test.go:798+)
exercises the interleaved-user-message scenario.

## What looks good

- Correct primitive: pre-indexing tool results by `ToolCallID` and
  pulling them forward to the assistant turn is the only sane way to
  satisfy adjacency without lying to the model.
- `placedResults` bookkeeping by **message ID** (not tool_call_id) is
  the right granularity — handles the case where a single Tool message
  carries multiple tool_results.
- The synthetic-error message is appended *after* real results in
  `buildToolResultsForCalls` (line 880), so the partial-results case
  (some real, some interrupted) is handled correctly.
- Test coverage extends real scenarios (interleaved user, late
  results) rather than just the happy path.

## Nits

1. `buildToolResultsForCalls` (line 850) always uses
   `range tcToResultMsgs[tc.ID]` — if the same tool_call_id has
   multiple result messages (which shouldn't happen but the map is
   `[]message.Message`), all of them get appended. Worth either:
   - Asserting `len(resultMsgs) <= 1` and logging a warning, OR
   - Using the **last** result (most recent retry) deterministically.
   Today it concatenates them, which would silently double-count.
2. The `slog.Warn` for synthetic injection (line 870) is unchanged
   from the old code — fine. But since this PR also changes the
   reorder semantics, consider adding a `slog.Debug` when a tool
   result message gets pulled forward (i.e. when placedResults marks
   it from the assistant pass). That would make production debugging
   of "why is my conversation reordered" tractable without a re-run.
3. The `tcToResultMsgs` map is built unconditionally for every
   `preparePrompt` call. For long sessions (thousands of messages)
   this is a non-trivial allocation. Consider gating behind a
   `provider.RequiresStrictAdjacency()` check — though for correctness
   it's fine to always do it.
4. Comment block (lines 802–805) is good; consider also documenting
   on `buildToolResultsForCalls` that the order of tool calls within
   the assistant message is preserved (the loop iterates
   `m.ToolCalls()`), which matters for some providers.

## Risk

Medium. Touches the hot path for every prompt assembly. The
non-adjacent test is great; consider also adding:
- A test where the same tool_call_id appears in the DB as a result
  that was already adjacent — confirms no double-emit.
- A test where two assistant messages each have tool calls and
  results land out of order across both turns.
