---
pr: 24428
repo: sst/opencode
sha: fa478297f13d87ae544034a07398d5a0bda7f336
verdict: merge-after-nits
date: 2026-04-26
---

# sst/opencode #24428 — preserve existing reasoning_content on second interleaved pass (#24146 follow-up)

- **Author**: claudianus
- **Head SHA**: fa478297f13d87ae544034a07398d5a0bda7f336
- **Size**: ~117 diff lines across `packages/opencode/src/provider/{provider.ts,transform.ts}`.

## Scope

PR #24146 fixed empty-`reasoning_content` preservation for DeepSeek by *unconditionally* setting `providerOptions[sdk][field] = reasoningText` on every interleaved-transform pass. The transform runs on every request, so on the second pass (after DB storage) content parts no longer have `reasoning` parts (they were extracted and stripped on the first pass), `reasoningText = ""`, and that empty string overwrites the previously-correct `providerOptions.reasoning_content` — DeepSeek then 400s with `"The 'reasoning_content' in the thinking mode must be passed back to the API."`. This PR (a) falls back to the existing field value when `reasoningText` is empty, (b) routes the field write through a dynamic `sdk` key instead of hardcoded `openaiCompatible`, (c) defaults `interleaved` for reasoning models in `fromModelsDevModel` when models.dev marks `reasoning: true` without an explicit `interleaved` config, and (d) adds a downstream `model.capabilities.reasoning` block that injects empty `reasoning_content` on *historical* assistant messages from before reasoning mode was enabled.

## Specific findings

- `packages/opencode/src/provider/transform.ts:208-214` — the actual fix:
  ```ts
  const existingField = msg.providerOptions?.[sdk]?.[field]
  const resolvedText = reasoningText || existingField || ""
  ```
  Correct shape: prefer the just-extracted reasoning, fall back to the previously-stored value, finally default to empty string. The `||` (rather than `??`) chain is right here because empty-string is *also* a value DeepSeek requires sent back, but we should never *prefer* empty-string over a non-empty existing value.
- `packages/opencode/src/provider/transform.ts:196` — `const sdk = sdkKey(model.api.npm) ?? "openaiCompatible"`. Good — the dynamic SDK key supports providers other than openaiCompatible (e.g., real Anthropic SDK routes for OpenRouter-routed DeepSeek) and the fallback preserves existing behavior.
- `packages/opencode/src/provider/transform.ts:179-181` — the deepseek-detection guard now checks both `model.api.id.includes("deepseek")` *and* `model.id.includes("deepseek")` to cover OpenRouter-routed DeepSeek where `api.id = "openrouter"` but `model.id = "deepseek/deepseek-r1"`. Right call.
- `packages/opencode/src/provider/transform.ts:229-262` — the new "inject empty reasoning_content for ALL assistant messages when reasoning is active but interleaved is not configured" block. This is correctness-preserving for DeepSeek's hard requirement, but the logic is duplicated for the array-content and string-content branches with only the content-rebuild differing. Extract a helper `injectEmptyReasoningContent(msg, sdk)` to avoid the two near-identical mutator branches drifting.
- `packages/opencode/src/provider/transform.ts:247-252` — when `msg.content` is a string, the patch converts it to a content-parts array `[{type: "text", text: msg.content}, {type: "reasoning", text: ""}]` *and* sets the providerOptions field. Adding a reasoning content-part with empty text is redundant with the providerOptions write — pick one. If the goal is for the next interleaved pass to find the reasoning part (so the providerOptions logic at `:200-227` runs), the part-injection is the right answer; in that case the providerOptions write here is the dead code. If the goal is the providerOptions write, the part-injection is dead code. Worth picking and removing the other.
- `packages/opencode/src/provider/provider.ts:1006` and `:1180` — `interleaved: model.interleaved ?? (model.reasoning ? { field: "reasoning_content" } : false)`. Sensible default — models.dev entries with `reasoning: true` but no explicit `interleaved` config now opt in to interleaved with the conventional field name. But there's no provider where `field` should be something other than `reasoning_content`? If, say, a future Anthropic-routed reasoning model uses `field: "thinking"`, this default is wrong. A comment pointing to the convention-source (or making this provider-conditional) would future-proof this.

## Risk

Medium. The core fix is right and well-explained. The duplicate empty-`reasoning_content` injection (providerOptions *and* synthetic reasoning content-part for the string-content branch) is a code-smell suggesting the author wasn't sure which path the downstream consumer reads. **Critical missing piece: zero tests in the diff slice.** A regression PR that fixes a "pass 2 overwrites pass 1" bug must ship a test asserting the round-trip preserves a non-empty value, plus the historical-message "reasoning enabled mid-conversation" case.

## Verdict

**merge-after-nits** — add (1) a regression test for the exact scenario in the PR body (first pass extracts `reasoning_content="…"`, second pass preserves it), (2) a test for the historical-message empty-injection case for both string and array content, (3) extract the duplicated injection branch into a helper, and (4) decide which path (`providerOptions` vs. synthetic reasoning content-part) is canonical for string-content messages and remove the other. The fix is right; the testing and code-shape work is small.
