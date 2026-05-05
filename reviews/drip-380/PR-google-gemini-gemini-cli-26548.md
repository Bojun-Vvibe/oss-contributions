# google-gemini/gemini-cli PR #26548 — fix(core): cache model routing decision in LocalAgentExecutor

- URL: https://github.com/google-gemini/gemini-cli/pull/26548
- Head SHA: `f173fe4f30bec2404aa335da9ebbb9c4af0a7532`
- Size: +107 / -18

## Summary

Caches the routing decision in `LocalAgentExecutor` so `auto`-model subagents resolve their concrete model **once** per executor lifetime instead of once per turn. Previously every turn called `router.route(...)` against the model router service (a non-trivial LLM-backed call in some configurations). For a 10-turn subagent run this was 10 redundant routing calls — and worse, transient routing failures partway through a run could silently switch the model mid-conversation.

## Specific findings

- `packages/core/src/agents/local-executor.ts:130` — adds `private cachedModelToUse?: string;` instance field. Scoped to a single executor (one subagent run), which is the right granularity — different subagent invocations should route independently.
- `packages/core/src/agents/local-executor.ts:954-977` — control flow:
  - `:954-955` `if (this.cachedModelToUse) { modelToUse = this.cachedModelToUse; }` — fast path on subsequent turns.
  - `:957-973` else-branch is the original try/catch routing logic, with `this.cachedModelToUse = modelToUse;` added at `:971` only on success. The catch branch at `:973-975` falls back to `DEFAULT_GEMINI_MODEL` and notably **does not** populate the cache — so a transient routing failure on turn 1 doesn't pin the fallback model for the entire run; turn 2 will retry. Correct policy.
- The `TODO(joshualitt)` comment at `:957-961` is preserved verbatim — good, the underlying policy question (consistency with main-agent routing failure handling) is unchanged by this fix.
- `packages/core/src/agents/local-executor.test.ts:2282-2362` — new 80-line test `should cache the routing decision across multiple turns`:
  - `:2284-2286` sets `definition.modelConfig.model = 'auto'` and `runConfig.maxTurns = 3`.
  - `:2288-2294` mocks `router.route` to return `'routed-model'`.
  - `:2336-2343` queues two model responses (LS_TOOL then COMPLETE_TASK_TOOL) so the run takes 2 turns.
  - `:2356` `expect(mockRouter.route).toHaveBeenCalledTimes(1)` — the load-bearing assertion.
  - `:2357-2371` asserts both `sendMessageStream` calls used `model: 'routed-model'`, pinning that the cached value is actually consumed.

## Concerns / nits

- **Cache never invalidates**: If `auto` resolves to `model-A` on turn 1 and the routing service's policy / model availability changes later in a long-running subagent run, the executor will keep using `model-A` forever. Probably acceptable (subagents are typically short-lived) but worth a one-line comment at the cache definition explaining the lifetime contract: "cache lives for the duration of one `run()` invocation; not invalidated".
- **No negative test**: missing a test that asserts a routing failure on turn 1 followed by a success on turn 2 produces a cached value from turn 2's success (not stuck on `DEFAULT_GEMINI_MODEL`). The implementation does the right thing but the contract isn't pinned.
- The cache is keyed on nothing — if `requestedModel` for some reason switches between `'auto'` and a concrete name across turns within the same executor run, the cache would still apply. Today the executor guarantees `requestedModel` is constant per run, so this is theoretical, but a one-line `// only valid because requestedModel is constant per executor lifetime` would harden the invariant.

## Verdict

`merge-after-nits` — the fix is correct and well-tested for the happy path; the missing piece is a doc comment on the cache lifetime/invalidation policy and ideally a negative-path test for "fail-then-succeed routing across turns".
