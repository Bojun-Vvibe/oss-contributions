# QwenLM/qwen-code #3775 — refactor(core): route side-query LLM calls through runSideQuery chokepoint

- URL: https://github.com/QwenLM/qwen-code/pull/3775
- Head SHA: `e65559d7256275695dca41972616c7c02b056961`
- Author: @tanzhenxin
- Stats: +1006 / -755 across 28 files

## Summary

Refactor that funnels every "auxiliary" LLM call (rewrite, summarization,
recap, title, follow-up suggestions, web-fetch, tool-use summary, ACP
rewrite, arena evaluation, etc.) through a single new `runSideQuery`
helper in `packages/core/src/utils/sideQuery.ts`. Call sites previously
reached for `config.getContentGenerator().generateContent(...)` directly
with handcrafted `{ model, config, contents }`; they now pass
`{ purpose, model, systemInstruction, config, abortSignal, contents }`
to `runSideQuery` and read `result.text`/`result.usage`.

## Specific feedback

- `packages/core/src/utils/sideQuery.test.ts` (+274/-76) is large and
  serves as the de-facto contract. Reviewer should confirm every
  callsite uses the **same** purpose-string convention (`'acp-rewrite'`,
  `'compression'`, `'title'`, etc.) so the new chokepoint can be used
  for routing/limit decisions later. Inconsistent purpose strings would
  defeat the entire point of the chokepoint.
- `packages/core/src/core/baseLlmClient.ts` (+113/-0) — adds the new
  surface in the base client. Confirm there's no second
  `BaseLLMClient`-style class outside this file that side-query callers
  might still reach for.
- `packages/cli/src/acp-integration/session/rewrite/LlmRewriter.ts:115-150`
  — the migration drops the explicit `thinkingConfig: { includeThoughts:
  false }` knob that the old code used. `runSideQuery` had better set
  this internally, otherwise rewriter output may now leak `thought`
  parts into `result.text`. The test mock at `LlmRewriter.test.ts:25-36`
  forwards `options` straight to `mockGenerateContent`, so this
  regression would NOT be caught by the unit test as written.
- `packages/cli/src/acp-integration/session/rewrite/LlmRewriter.ts:140`
  — `abortSignal: signal ?? new AbortController().signal` constructs a
  throwaway `AbortController` per call when the caller didn't pass a
  signal. Functionally fine but allocates needlessly; `runSideQuery`
  should accept `AbortSignal | undefined` and synthesize the
  controller once internally.
- `packages/core/src/services/chatCompressionService.ts` (+22/-20)
  and `chatCompressionService.test.ts` (+135/-248) — the compression
  service test shrinks meaningfully. Verify the deleted assertions
  (e.g. on token counts in usage events) are still covered elsewhere
  — compression cost-tracking is the kind of telemetry that quietly
  goes dark in a chokepoint refactor.
- `packages/core/src/agents/arena/ArenaManager.ts` (+15/-44) — the
  arena loop relied on raw `Candidate[]` shapes from `generateContent`.
  Confirm `runSideQuery` exposes enough of the underlying
  `GenerateContentResponse` (or returns a richer structured object)
  for callers that need more than `.text`/`.usage`. If `result` is
  text-only, the arena code may have been silently truncated.
- `packages/cli/src/services/insight/generators/DataProcessor.test.ts:+20/-1`
  — the test mock had to be updated to return a fully-populated
  schema-validated object, suggesting the new pipeline is stricter
  about partial responses. That's a behaviour change worth noting in
  the PR description: pre-refactor, partial LLM responses degraded
  gracefully; post-refactor, schema-rejected partials become hard
  failures.
- `packages/core/src/utils/internalPromptIds.ts:+10/-3` — adds the new
  purpose-string identifiers. Make sure these are kept in sync with
  whatever billing/telemetry sink consumes them.
- 28-file refactor with no behaviour-change CHANGELOG entry is risky;
  this is the kind of PR a future bisect will land on for "side-query
  X stopped working".

## Verdict

`needs-discussion` — the chokepoint direction is right, but three
behavioural deltas surfaced in the diff (lost `thinkingConfig` in
rewriter, stricter schema enforcement in DataProcessor, possible
loss of structured-response shape for arena) need to be answered
in the PR thread before merge. None are blocking-by-themselves,
but together they hint that `runSideQuery`'s contract is narrower
than the old call sites assumed.
