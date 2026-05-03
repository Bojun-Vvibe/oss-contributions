# QwenLM/qwen-code PR #3735 — fix(core): auto-compact subagent context to prevent overflow

- URL: https://github.com/QwenLM/qwen-code/pull/3735
- Head SHA: `07480e3db2524b24070d4f35b8901e9c2740f0c9`
- Author: tanzhenxin
- Repo: QwenLM/qwen-code

## Summary

Pushes context-compression into `GeminiChat.tryCompress` and refactors
`GeminiClient.tryCompressChat` to a thin delegating wrapper, then surfaces
subagent compactions via the agent runtime's debug log so subagent compaction
events appear alongside main-session ones. Also tightens
`findCompressSplitPoint` semantics for in-flight trailing function calls.

## Specific notes

- `packages/core/src/agents/runtime/agent-core.ts:543-552` — new
  `streamEvent.type === 'compressed'` branch logs `subagent=<id>
  round=<turnCounter> tokens orig -> new`. Pure debug log, no behavior
  change for downstream UI. Good. Could promote to a dedicated UI event
  later if subagent compactions need user surfacing.
- `packages/core/src/core/client.ts` — `tryCompressChat` now delegates to
  `chat.tryCompress`. The client.test.ts `beforeEach` (line 414+) stubs
  `client.tryCompressChat` to NOOP for unrelated tests so the shared mock
  chats (which lack `tryCompress`) don't crash. That's the right test-
  hygiene fix; flagged here because it's load-bearing for the PR's huge
  test churn (~580 lines moved out of `client.test.ts` into
  `chatCompressionService.test.ts` / `geminiChat.test.ts`).
- `client.test.ts:240-244` — semantics change in
  `findCompressSplitPoint`: with a trailing m+functionCall, the new
  expectation is `splitPoint = 3` (compress everything except the
  trailing fc) vs the old `2` (back off to the prior pair). Comment
  explains "in-flight fallback". This is a behavior change worth
  calling out in release notes — anyone subscribing to compression
  events will see slightly larger compacted ranges in this edge case.
- New test `seeds resumed chat with replayed prompt token count`
  (client.test.ts:425-446) — good regression coverage for resumed
  sessions. Verifies `getLastPromptTokenCount === 123_456` after init.
- ~600 LOC of test deletions in `client.test.ts` are migrated to the
  service-level tests. Spot-check that no scenarios were dropped:
  the `tryCompressChat` parametrized matrix (originalTokenCount,
  compressionInputTokenCount, etc.) needs to land in
  `chatCompressionService.test.ts` with equivalent coverage.

verdict: merge-after-nits
