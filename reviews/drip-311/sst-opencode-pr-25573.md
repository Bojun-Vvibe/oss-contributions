# sst/opencode #25573 — fix(cf-ai-gateway): route provider options through openaiCompatible key (#24432)

- PR: https://github.com/sst/opencode/pull/25573
- Author: NathanDrake2406
- Head SHA: `a98026011a29bc19ed5907a1d1cbb72cca7c5cd3`
- Updated: 2026-05-03T09:25:29Z

## Summary
Fixes #24432: `reasoning_effort` and other provider options were being dropped on the floor when routed through Cloudflare AI Gateway because `ai-gateway-provider/unified` wraps `createOpenAICompatible({ name: "Unified" })`, and the SDK now expects the `openaiCompatible` (camelCase) key, not the deprecated `openai-compatible` form. Adds the `"ai-gateway-provider"` case to `sdkKey()` returning `"openaiCompatible"` (`packages/opencode/src/provider/transform.ts:42-50`), refactors the OpenAI effort-tier computation into a shared `openaiReasoningEfforts(apiId, releaseDate)` helper (`transform.ts:444-457`), wires it into the new `"ai-gateway-provider"` `variants` case (`transform.ts:516-530`), and adds a substantial e2e regression test that stubs only the network boundary (`packages/opencode/test/provider/cf-ai-gateway-e2e.test.ts`, +110 lines).

## Verdict
`merge-as-is`

## Observations
- `packages/opencode/src/provider/transform.ts:42-50`: the comment correctly identifies that the SDK's `compatibleOptions` parser accepts four spellings and only `"openai-compatible"` triggers a deprecation warning. Good documentation of *why* `openaiCompatible` was chosen.
- `packages/opencode/src/provider/transform.ts:455`: the `GPT5_FAMILY_RE = /(?:^|\/)gpt-5(?:[.-]|$)/` is a real correctness improvement over the old `id.includes("gpt-5-") || id === "gpt-5"` check at the removed `transform.ts:~600`. The old check would have missed `openai/gpt-5.4` (no dash) and false-matched `gpt-50`. New regex anchors properly. The PR notes this anti-false-match for `gpt-5o` but `gpt-5o` would actually match `[.-]|$` only if it ended at `5` — `gpt-5o` doesn't match because `o` is neither `.` nor `-` nor end. Correct.
- `packages/opencode/src/provider/transform.ts:460-470`: `openaiReasoningEfforts` returns `null` for `gpt-5-pro` / `openai/gpt-5-pro` — preserves the prior "no variants" behavior. Codex variants special-case `5.2`/`5.3` for `xhigh`. Logic is identical to what was inlined in the old `@ai-sdk/openai` case, so no regression.
- `packages/opencode/src/provider/transform.ts:516-530`: the new `"ai-gateway-provider"` case checks `model.api.id.startsWith("openai/")` to decide whether to use the extended OpenAI effort set; otherwise falls back to `WIDELY_SUPPORTED_EFFORTS`. Reasonable since CF AI Gateway translates `reasoning_effort` to native upstream controls (e.g. Anthropic thinking budgets) and `low/medium/high` is the universal set.
- `packages/opencode/src/provider/transform.ts:516-530`: variant value is `{ reasoningEffort: effort }` (camelCase), matching the new `sdkKey` choice — they have to agree, and they do.
- `packages/opencode/test/provider/cf-ai-gateway-e2e.test.ts`: the test stubs only `globalThis.fetch` for `gateway.ai.cloudflare.com/*` and asserts on `captured.outerBody[0].query` (the actual request body the gateway forwards upstream). This is exactly the right boundary — earlier mocks at the SDK layer are why this bug shipped.
- `packages/opencode/src/provider/transform.ts:650-664`: the `@ai-sdk/openai` case was rewritten to call the shared helper. The `iife()` wrapper is gone, which is a small readability win. Behavior should be byte-identical for OpenAI-direct models.

Excellent PR. Clear bug, focused fix, root-cause explanation in comments, real e2e test that would have caught the regression. Merge as-is.
