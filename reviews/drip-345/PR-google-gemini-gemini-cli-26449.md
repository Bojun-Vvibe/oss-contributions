# google-gemini/gemini-cli#26449 — fix(core): isolate subagent thread context

- PR ref: `google-gemini/gemini-cli#26449` (https://github.com/google-gemini/gemini-cli/pull/26449)
- Head SHA: `377e571a536c79967bac68e9d5669f0f94d2010a`
- Title: fix(core): isolate subagent thread context
- Verdict: **merge-after-nits**

## Review

The semantic change is in `packages/core/src/agents/local-executor.ts:310-316`: instead
of pulling `parentPromptId` off `mockConfig.promptId` (mutable global-ish state), the
executor now derives `agentId` from the constructor-passed `parentCallId`, sanitizes it
to `[a-zA-Z0-9_-]+` so it's safe as a path/log token, and concatenates a 6-char random
suffix. The fallback (no parent call id → pure random) preserves the old shape for
top-level executors. Sound.

The test rewrite at `packages/core/src/agents/local-executor.test.ts:693-707` now wraps
the `LocalAgentExecutor.create` call in `runWithToolCallContext({ callId: parentCallId,
schedulerId: 'test-scheduler' }, ...)` and asserts `executor['agentId']` *contains* the
parent call id. That's the right shape of assertion — substring rather than equality —
because of the random suffix. The deletion of the `Object.defineProperty(mockConfig,
'promptId', ...)` setup also confirms the intent: this code path is no longer reading
from config-as-context, only from explicit constructor wiring. Good decoupling.

Nits:
1. The sanitization regex `/[^a-zA-Z0-9_-]/g` at `local-executor.ts:313` will collapse
   any sequence of disallowed chars into `_`, which means two distinct `parentCallId`s
   that differ only in punctuation would alias to the same prefix. Unlikely in practice
   (call ids are usually UUID-shaped), but worth a one-line comment noting the
   collision is acceptable because the trailing random suffix still makes `agentId`
   unique.
2. `Math.random().toString(36).slice(2, 8)` gives ~6 chars of base36 ≈ 31 bits of
   entropy. Fine for collision-resistance within a single CLI session; not fine for
   anything cryptographic. Recommend a comment to that effect at line 314 so this
   doesn't get confused with a security identifier later.
3. No test for the sanitization itself — a `parentCallId` like `'foo/bar:baz'` should
   become `'foo_bar_baz-XXXXXX'`. Worth one parameterized case.
