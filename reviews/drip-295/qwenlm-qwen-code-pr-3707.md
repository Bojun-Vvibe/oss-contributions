# QwenLM/qwen-code PR #3707 — fix(core): per-agent ContentGenerator view via AsyncLocalStorage

- **Repo:** QwenLM/qwen-code
- **PR:** #3707
- **Head SHA:** `be9ba5f59a3567f5cdd4888bc0fca4e7ac49baea`
- **Author:** tanzhenxin
- **Title:** fix(core): per-agent ContentGenerator view via AsyncLocalStorage
- **Diff size:** +359 / -183 across 14 files
- **Drip:** drip-295

## Files changed (substantive subset)

- `packages/core/src/agents/runtime/agent-context.ts` (+63, new) — defines `AgentContext { agentId?, runtimeView? }` over a `node:async_hooks` `AsyncLocalStorage`. Exposes `runWithAgentContext(agentId, fn)`, `runWithRuntimeContentGenerator(view, fn)`, `getCurrentAgentId()`, `getRuntimeContentGenerator()`. Both `runWith*` helpers spread the existing store first (`...current`) so nested wrapping at different layers preserves earlier fields.
- `packages/core/src/agents/runtime/agent-context.test.ts` (+163, new) — exercises three describe blocks: `(agentId)`, `(runtimeView)`, `(merging)`. The merging tests at lines 295-317 are the load-bearing ones (nested `runWithAgentContext` inside `runWithRuntimeContentGenerator`).
- `packages/core/src/config/config.ts` (+12/-4) — three `Config` getters now defer to ALS:
  - `getContentGenerator()` → `getRuntimeContentGenerator()?.contentGenerator ?? this.contentGenerator`
  - `getContentGeneratorConfig()` → `... ?? this.contentGeneratorConfig`
  - `getModel()` and `getAuthType()` route through `getContentGeneratorConfig()` so they pick up the override transitively.
- `packages/core/src/agents/runtime/agent-core.ts` (+36/-7) — `AgentCore` gains `readonly runtimeView?: RuntimeContentGeneratorView`, and the run-loop wraps execution in `runWithRuntimeContentGenerator(this.runtimeView, fn)` when set (line ~467).
- `packages/core/src/agents/backends/InProcessBackend.ts` (+19/-12) and its test (+13/-21) — InProcess backend wires the runtime view onto the spawned `AgentCore`.
- `packages/core/src/subagents/subagent-manager.{ts,test.ts}` (+30/-30 net) — subagent manager no longer monkeypatches `Config.getContentGenerator` overrides; relies on the ALS view instead.
- `packages/core/src/tools/agent/agent-context.ts` (-45) and its test (-53) — **deleted**: the old per-agent `agentContext` proxy that overrode `getContentGenerator` on a Config wrapper is gone.
- `packages/core/src/tools/agent/agent.ts` (+6/-3), `packages/core/src/tools/agent/agent.test.ts` (+4/-7) — agent tool launches now publish their id via `runWithAgentContext` and inherit the parent's runtimeView via the ALS spread semantics.

## Specific observations

- `agent-context.ts:33-44` — `AsyncLocalStorage` propagation across `await` boundaries is exactly what's needed here, and the spread-then-overwrite pattern (`{ ...current, agentId }`) means nested helpers compose correctly: setting `runtimeView` at the outer layer and `agentId` at the inner layer leaves both visible. The tests at line 295-317 pin this. Note however that **breaking out of an awaited Promise via `setImmediate`/`setTimeout` callbacks scheduled with the wrong context** can lose the store. If any tool implementation captures a callback with `setTimeout(cb, 0)` after a `Config.getContentGenerator()` call, the captured callback will see the original (parent) Config getters — which is exactly the bug this PR is trying to fix in reverse. Worth a brief note in the doc comment about ALS lifetime.
- `config.ts:1187-1192` and `:1359-1366` — the precedence is `ALS view > stored field`. That's the correct direction for this fix, but it inverts the previous mental model where `Config` was the source of truth. Any test or downstream consumer that does `expect(config.getContentGenerator()).toBe(specificInstance)` from outside an ALS scope still works (falls through to `this.contentGenerator`), but anyone asserting the ALS-active variant must wrap the assertion in `runWithRuntimeContentGenerator(..., async () => { ... })`. The deletion of `tools/agent/agent-context.test.ts` (-53) suggests the prior assertions are gone, but reviewers should grep for any remaining `getContentGenerator()` assertions in unrelated test files to confirm none silently switched semantics.
- `config.ts:1378-1383` — `getModel()` now resolves through `getContentGeneratorConfig()`, which itself routes through ALS. Before this PR `getModel()` read `this.contentGeneratorConfig?.model` directly, so it was synchronously model-of-this-Config. After this PR `getModel()` is *contextual*. That's a behavior change for any caller that captured a Config reference and called `getModel()` outside the agent's ALS frame expecting the parent model. The PR description should call this out explicitly — it's the load-bearing change of the whole PR but is a one-line diff.
- `agent-core.ts:248` (the `runWithRuntimeContentGenerator(this.runtimeView, fn)` wrap) — confirm that the wrap happens **before** any tool construction. Tools that capture `this.config.getContentGenerator()` in their constructor (the entire reason for this PR per the doc comment) need to be constructed *inside* the ALS frame, not outside; otherwise the constructor still sees the parent CG. The diff shows the wrap at the run-loop boundary; if tool instances are reused across runs, only the *call* pattern (where `getContentGenerator` is called from inside an ALS frame each turn) protects them, not construction. Worth a test that explicitly constructs a tool outside the frame, then calls it inside, and asserts the inner CG is used.
- `agent-context.test.ts:218-228` — concurrent test runs `runWithAgentContext('a', ...)` and `runWithAgentContext('b', ...)` in parallel and asserts isolation. Good — this is the canonical ALS-isolation test and it's necessary because parallel sub-agent execution is the primary use case.
- `agent-context.test.ts` does **not** cover the case of an exception thrown inside the wrapped fn. ALS does the right thing automatically (the store is restored on unwind), but the absence of a "throws inside wrap, then `getCurrentAgentId()` returns null afterward" test leaves a small gap.
- The deletion of 53+45=98 lines from `tools/agent/agent-context.{ts,test.ts}` is the right cleanup — the old monkeypatch-the-Config approach is genuinely replaced by this ALS frame, not duplicated.
- `subagent-manager.ts` (+18/-26 net) — the diff replaces `override.getContentGenerator = (): ContentGenerator => agentGenerator` style monkeypatching. Confirm no callers in third-party plugins or external consumers were relying on the old override-on-Config-clone pattern. If yes, this is a quiet breaking change to a public-ish API.
- Performance: `AsyncLocalStorage` has measurable overhead in hot paths under Node 18-20. Every `getContentGenerator()` / `getContentGeneratorConfig()` call now incurs a `storage.getStore()` lookup. For an agent loop that calls these on every turn this is fine (microseconds), but if any tight inner loop in a tool implementation hits these, profile first.

## Verdict: `merge-after-nits`

Architecturally clean and the test coverage is solid (especially the parallel-isolation case). Three things to address: (1) call out the `getModel()` semantics change in the PR body — it's a one-line but consequential behavior shift; (2) add a test for "tool constructed outside ALS frame, called inside ALS frame, sees the inner CG" — that's the precise scenario the doc comment claims to fix; (3) optional: an "exception inside wrap restores store" test to lock in ALS unwind semantics. The code itself is the right shape — replacing the monkeypatch with an ALS frame is the canonical idiom for this problem.
