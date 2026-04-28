# QwenLM/qwen-code#3707 — fix(core): per-agent ContentGenerator view via AsyncLocalStorage

- **PR**: https://github.com/QwenLM/qwen-code/pull/3707
- **Author**: @tanzhenxin
- **Head SHA**: `be9ba5f59a3567f5cdd4888bc0fca4e7ac49baea`
- **Base**: `main`
- **State**: OPEN
- **Scope**: +359 / -183 across 14 files

## Summary

Restructures how a sub-agent overrides the parent session's `ContentGenerator` (CG). The old approach at `InProcessBackend.ts` shadowed four methods on the parent `Config` via `Object.create(base)` + ad-hoc `override.getContentGenerator = () => agentGenerator` (and `getContentGeneratorConfig`, `getAuthType`, `getModel`). That worked for code that asked the *override* Config but not for tools that captured the *parent* Config at construction time and then ran inside a sub-agent context — those tools kept seeing the parent CG. The fix introduces an `AsyncLocalStorage<AgentContext>` with `runtimeView?: RuntimeContentGeneratorView`, has the agent's reasoning loop wrap its body in `runWithRuntimeContentGenerator(view, fn)`, and changes `Config.getContentGenerator{,Config}()` to consult the ALS frame *before* falling back to its own field. Now every CG-related getter resolves to the agent's view inside the agent's run, regardless of which `Config` object the caller captured. This is the right shape for the "tools captured parent Config but should see agent CG" bug and is structurally cleaner than per-instance method shadowing.

## Diff anchors

- `packages/core/src/agents/runtime/agent-context.ts` (new file, ~50 lines) — the load-bearing module:
  - `interface RuntimeContentGeneratorView { readonly contentGenerator; readonly contentGeneratorConfig }` at `:353-356` — minimal view shape that downstream getters need.
  - `interface AgentContext { agentId?; runtimeView? }` at `:358-361` — single ALS payload carrying both the existing `agentId` slot and the new `runtimeView` slot. Sharing one ALS instance is correct (avoid two ALS frames racing); `agentId` already lived here as a separate `subagentNameContext` and the merge into a single store is a nice cleanup.
  - `runWithRuntimeContentGenerator(view, fn)` at `:373-379` — uses `{ ...current, runtimeView: view }` to *layer* the view onto whatever frame is already active. Load-bearing because nested agent runs must shadow correctly without losing outer `agentId`.
  - `getRuntimeContentGenerator()` at `:384-388` — returns the view or `undefined`. The undefined-when-outside-frame contract is what makes the `??` fallback at the call sites correct.
- `packages/core/src/config/config.ts:1190-1192` — `getContentGenerator()` now returns `getRuntimeContentGenerator()?.contentGenerator ?? this.contentGenerator`. The view-first / field-fallback ordering is the right invariant: if the agent published a view, all callers see it; otherwise, callers see the construction-time CG.
- `packages/core/src/config/config.ts:1361-1364` — symmetric change for `getContentGeneratorConfig()`.
- `packages/core/src/config/config.ts:1380-1382` — `getModel()` is rewritten to consult `this.getContentGeneratorConfig()?.model` instead of the raw `this.contentGeneratorConfig?.model` field. This is load-bearing: without it, `getModel()` would skip the ALS layer and report the parent's model even when an agent view is active.
- `packages/core/src/config/config.ts:2313` — `getAuthType()` similarly routed through `this.getContentGeneratorConfig()` instead of the raw field. Four-getter symmetry (CG / CG-config / model / auth-type) closes the surface.
- `packages/core/src/agents/runtime/agent-core.ts:194, 251, 263, 444-460` — `AgentCore` gains a `readonly runtimeView?` field, an extra constructor parameter, and a `withRuntimeView<T>(fn)` helper that conditionally wraps in `runWithRuntimeContentGenerator(this.runtimeView, fn)`. The reasoning loop at `:441` now nests `subagentNameContext.run(name, () => withRuntimeView(inner))` — inner-most layer is the runtime view, outer is the subagent-name context.
- `packages/core/src/agents/runtime/agent-core.ts:925-930` — second `withRuntimeView` site for the resumed-tool-after-confirmation path. The inline comment "UI invokes this from its own async chain (outside the reasoning-loop ALS frame), so re-enter the agent's view before the resumed tool body runs" is exactly right and is the kind of subtle correctness point that needs the comment to survive future edits.
- `packages/core/src/agents/backends/InProcessBackend.ts:355-377` — `createPerAgentConfig` now returns `{ config, contentGenerator?, runtimeView? }` instead of poking `override.getContentGenerator = ...` shadows onto the prototype. The five `override.getX = ...` lines deleted at `:408-413` are the symptoms the new architecture eliminates.
- `packages/core/src/agents/runtime/agent-context.test.ts` (new test file, 163 lines) — unusually principled coverage matrix:
  - Outside-frame returns `null`/`undefined` for both agentId and runtimeView (`:235-236`, `:249-250`).
  - Inside-frame exposure (`:255-260`).
  - Async-await propagation (`:262-268`) — the canonical ALS smoke test.
  - Nested-frame shadowing (`:270-279`).
  - Concurrent-frame isolation via `Promise.all` (`:281-294`) — load-bearing for catching a future regression that swaps ALS for a global.
  - Sibling-run isolation for views (`:262-275`).
  - **The two cross-axis tests at `:296-322` are the highest-value cells**: `runtimeView wrap preserves agentId from outer frame` and `agentId wrap preserves runtimeView from outer frame`. These pin the `{ ...current, X: Y }` spread invariant — without these, a future contributor "simplifying" to `storage.run({ X: Y }, fn)` would silently lose the outer slot and break the layered ALS model.
- `packages/core/src/agents/backends/InProcessBackend.test.ts:495-595` — three test rewrites flip from "assert `agentContext.getContentGenerator()` returns agent" to "assert `lastCall![8].contentGenerator` is the view payload". The `lastCall![8]` index encodes the new constructor arity for `AgentCore`; brittle if reviewer cares about argument-positional tests, but accurate to the current shape.

## What I'd push back on (nits)

1. **`lastCall![8]` is positional-argument-magic-number**. With 9 constructor arguments, an off-by-one in a future signature change makes the tests fail in a confusing way. Either use named-options bag for `AgentCore` constructor, or destructure `[, , , , , , , , runtimeView]` so the position is named at the call site.
2. **The `models/modelRegistry.ts:176-188` change is in this diff but barely visible from the title**. The 9-line `generationConfig = { ...(config.generationConfig ?? {}) }` clone + auto-fill modalities path is unrelated to ALS plumbing. Either split out or call out in PR body.
3. **No test for the "tool captured parent Config" failure mode the PR is about**. The new tests pin the ALS primitives correctly, but there's no integration test that constructs a tool with a parent Config, runs it under an agent's `withRuntimeView`, and asserts the tool's `Config.getContentGenerator()` returns the agent CG. That's the whole motivation; it deserves a direct test cell.
4. **`subagentNameContext` and the new `agentId` slot on `AgentContext` are now near-duplicates**. The PR merges runtimeView into the new ALS but leaves `subagentNameContext` as a separate ALS instance (line 444 `subagentNameContext.run(this.name, ...)`). Two ALS frames are unnecessary; the merge is half-done. Recommend follow-up to consolidate.
5. **`AgentCore` constructor now takes 9 positional args**. Past 5 it's already a code-smell; 9 is unfriendly. Options bag conversion is a separate PR but worth queuing.

## Verdict

**merge-after-nits** — the architectural shape is correct (ALS-published view replaces ad-hoc method shadowing on a `Object.create(base)` proxy, which closes the "tool captured parent Config" hole), the cross-axis tests at `:296-322` prove the author understood the layered-spread invariant, and the four-getter symmetry across `config.ts` (CG, CG-config, model, auth-type) closes the API surface. Recommend (a) destructure `lastCall![8]` to a named binding, (b) split out the `modelRegistry.ts` modalities change, (c) add the integration test for the parent-Config-captured-by-tool case.

Repo coverage: QwenLM/qwen-code (sub-agent runtime architecture).
