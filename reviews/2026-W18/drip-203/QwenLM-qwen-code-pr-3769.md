# QwenLM/qwen-code #3769 — fix(core): isolate fast model side queries

- **Author:** LaZzyMan
- **SHA:** `6371959`
- **State:** OPEN
- **Size:** +168 / -58 across 18 files (4 production sources, 1 chat-history
  helper, 8 test files, 5 service refactors)
- **Verdict:** `merge-after-nits`

## Summary

Three mostly-independent changes shipped as a single PR, all framed by the
"fast model side queries should not pollute the main turn" theme:

1. **Per-model `ContentGenerator` cache** for side queries that target a
   model different from the active turn model. Both `BaseLlmClient`
   (`baseLlmClient.ts:69-94`) and `GeminiClient`
   (`client.ts:131-217`) gain a `Map<string, ContentGenerator>` cache and a
   `getContentGeneratorForModel(model)` helper that returns the active
   generator for the current turn model and constructs (and caches) a
   per-model generator via `buildAgentContentGeneratorConfig(this.config,
   model, { authType: ...! })` + `createContentGenerator(...)` for any
   other model. Side-query call sites in `relevanceSelector.ts:91-101`,
   `sessionRecap.ts:74`, `sessionTitle.ts:147-150`,
   `toolUseSummary.ts:285-291`, and `nextSpeakerChecker.ts` all now route
   through this helper.

2. **`thinkingConfig: { includeThoughts: false }`** added to every
   side-query call site so the fast model can't waste tokens emitting
   `<thinking>` traces that the side-query never reads. Visible in
   `renameCommand.ts:84`, `relevanceSelector.ts:99`, `sessionRecap.ts:75`,
   `sessionTitle.ts:150`, `toolUseSummary.ts`, and the corresponding test
   assertions (e.g. `renameCommand.test.ts:224-226`,
   `sessionTitle.test.ts:142-149` adds explicit
   `expect(callOpts.config.thinkingConfig).toEqual({ includeThoughts:
   false })` lock).

3. **Auto-memory injection moved from "system reminder concatenated to
   user message" to "history insert before last user message"** — new
   helper `GeminiChat.insertBeforeLastUserMessage(content)` at
   `geminiChat.ts:787-795` (correctly defensive: appends if last role
   isn't user) and call site refactor at `client.ts:868-880` that uses
   `Promise.race([relevantAutoMemoryPromise, EMPTY_RELEVANT_AUTO_MEMORY_RESULT])`
   to keep the inject-or-not decision non-blocking. Test refactor at
   `client.test.ts:1391-1424` updates the assertion shape from
   `expect.arrayContaining(['## Relevant memory\n\n...'])` to
   `expect(mockChat.insertBeforeLastUserMessage).toHaveBeenCalledWith(...)`.

## Reasoning

Each piece is independently sensible:

1. **Per-model generator cache is the right shape for the side-query
   problem.** Today every side query against the fast model would either
   silently inherit the main model's provider config (wrong: provider
   may differ entirely) or pay the cost of building a fresh generator
   per call (slow). The `Map<string, ContentGenerator>` cache keyed by
   model id with the "current model returns the existing generator" fast
   path at `baseLlmClient.ts:74-77` and `client.ts:194-196` is the
   minimum viable solution. The non-null assertion on `authType!` at
   `:84` and `:206` is justified because the only call site that builds
   a generator already requires a configured `authType` to exist for
   the main client to have been constructed.

2. **`thinkingConfig: { includeThoughts: false }` for side queries is
   genuinely cost-saving.** Every test now locks this — including
   `relevanceSelector.test.ts:65-71` which also locks the new
   `model: 'qwen-fast'` routing via `config.getFastModel() ??
   config.getModel()` — so a future revert that re-enables thoughts on
   side queries will fail loudly.

3. **`insertBeforeLastUserMessage` is the cleaner injection point.**
   The previous shape concatenated the relevant-memory prompt as a
   "system reminder" appended to the user's actual message text, which
   muddled the conversation history (the user's message in the
   transcript looked longer than what they typed) and prevented turn
   compression from correctly identifying the synthetic prefix. The new
   shape inserts a separate `{role: 'user', parts: [{text: ...}]}` entry
   in the history *before* the last user message, which preserves the
   user's authored text intact. The defensive "if last isn't user, just
   push" branch at `:792-794` is correct.

Three nits worth a follow-up before merge:

1. **The PR is doing three things at once.** The per-model cache, the
   thinking-config disable, and the auto-memory insertion-point change
   are independently useful and independently testable, but bundling
   them makes bisection harder if any one introduces a regression. The
   commit-history shows the author did separate them cleanly into
   commits, so a maintainer could request a split into 3 PRs without
   losing work.

2. **`Promise.race([relevantAutoMemoryPromise ??
   Promise.resolve(EMPTY), Promise.resolve(EMPTY)])` at `client.ts:866-870`
   is a clever non-blocking pattern but the rationale isn't documented.**
   The intent ("inject auto-memory if it's already resolved by the time
   we start streaming, otherwise skip this turn") needs a one-line
   comment, otherwise the next reader will assume the `Promise.race`
   is a bug — it looks like the EMPTY result will always win the race
   for a synchronous resolved promise (which is the actual intent, but
   it's nonobvious).

3. **The `contentGeneratorsByModel` cache has no eviction strategy.**
   For a long-running session that switches between many fast models
   (a multi-provider session), the map will grow unbounded. The cache
   is per-`GeminiClient` instance so it lives for the session, not
   forever, but a `LRUMap` with a sane upper bound (say 8) would cost
   nothing and prevent the pathological case. Worth a follow-up issue.

Net: three useful changes, all locked by tests, all matching the existing
code conventions. Worth shipping after either splitting or at minimum
adding the `Promise.race` rationale comment.
