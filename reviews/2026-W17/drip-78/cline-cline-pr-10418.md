# cline/cline PR #10418 — added deepseek-v4 models

- **PR:** https://github.com/cline/cline/pull/10418
- **Head SHA:** `87dca6c9a9ed0cde6dcc5d69c68de5d49babe0ff`
- **Files:** 2 (+25 / -2)
- **Verdict:** `request-changes`

## What it does

Adds `deepseek-v4-pro` and `deepseek-v4-flash` to `src/shared/api.ts:2102-2125`
and gates a new `isDeepseekV4` flag in the DeepSeek handler at
`src/core/api/providers/deepseek.ts:84`. Default model is changed from
`"deepseek-chat"` → `"deepseek-v4-pro"` at `src/shared/api.ts:2101`.

## Specific reads

- `deepseek.ts:84` — `const isDeepseekV4 = model.id.startsWith("deepseek-v4")` is added but **never referenced anywhere else in the diff**. Dead variable.
- `deepseek.ts:97` — temperature gate stays `model.id === "deepseek-reasoner" ? {} : { temperature: 0 }`. The new comment claims "V4 models have thinking on by default" but the V4 models still get `temperature: 0`. If V4 models also need temperature suppressed (the comment implies they do), the condition should be `model.id === "deepseek-reasoner" || isDeepseekV4 ? {} : { temperature: 0 }`. Otherwise the dead `isDeepseekV4` flag is the smoking gun for an unfinished change.
- `api.ts:2102` — default flipped to `"deepseek-v4-pro"`. **This is a behavior change for every existing user with the DeepSeek provider configured but no explicit model pin** — they'll silently switch to v4-pro on next session, which has different pricing, different reasoning behavior, and different cache semantics. Should be a separate PR with a release-notes bump or at minimum should respect any pre-existing user config such that only fresh installs get the new default.
- `api.ts:2104-2114` (v4-pro) and `2115-2125` (v4-flash) — `inputPrice: 0` with comment "technically there is no input price, it's all either a cache hit or miss" is creative but wrong shape: any caller computing `inputPrice * inputTokens` will under-bill. The `cacheReadsPrice` + `cacheWritesPrice` model assumes 100% of input goes through one of the two. The first-ever message in a session has no cache hit *or* miss-from-prior — it's a cold input. Confirm the DeepSeek API actually charges all uncached input as a cache *miss* and not as a separate "cold" tier.
- `api.ts:2104-2125` — `supportsImages: false`, `supportsPromptCache: true`, `supportsReasoning: true`, `maxTokens: 384_000`, `contextWindow: 1_000_000`. The 1M context claim should link the source (DeepSeek docs URL) for posterity; existing entries do.

## Why request-changes

1. **Dead `isDeepseekV4` variable + claim/code mismatch** at `deepseek.ts:84,97`: either wire it into the temperature gate or drop both the variable and the misleading comment. As-is the PR is incoherent: it adds a guard for V4-models-have-thinking-on but doesn't actually skip `temperature: 0` for them, which is exactly the bug the comment implies it's fixing.
2. **Default-model flip is a silent breaking change** at `api.ts:2101`: split into a separate PR or migrate gracefully (read existing user setting first, only apply new default for fresh provider configs). Existing users will be surprised by a different model + different price + different reasoning style on next launch.
3. **`inputPrice: 0` with a "really it's cache" comment** at `api.ts:2110, 2121`: the cost-tracking layer needs a non-zero `inputPrice` for cold-cache requests, or an explicit `inputPrice: null` / sentinel that the cost path knows to special-case. Setting it to `0` will silently hide DeepSeek spend from any usage dashboard.
4. **No tests**: a model-table change is mechanical, but the `deepseek.ts` handler change is behavioral. At minimum add a unit test asserting that V4 models get/don't get `temperature: 0` per intended semantics.
5. **No source link**: add the DeepSeek pricing/docs URL inline as comments above each model entry, mirroring the existing `// https://api-docs.deepseek.com/quick_start/pricing` precedent at `api.ts:2099`.
6. **`isDeepseekReasoner` no longer used after the change?** Spot-check that the existing `isDeepseekReasoner` branch at `deepseek.ts:86-91` is still correct for v4-pro (which is also a reasoning model per `supportsReasoning: true`); v4-pro probably needs the same `convertToOpenAiMessages` reasoning-conversion path as `deepseek-reasoner`, but currently it falls through to the non-reasoner branch.
