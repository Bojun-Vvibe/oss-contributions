# anomalyco/opencode#25985 — fix: inject cache_control on content blocks for openai-compatible Bedrock proxies (Bifrost, LiteLLM)

- URL: https://github.com/anomalyco/opencode/pull/25985
- PR: #25985
- Author: KTS-o7 (Krishnatejaswi S)
- Head SHA: `8a4d6d2180c75e406540b29f66d04b83be3b9cf5`
- State: OPEN  | +213 / -4

## Summary

Closes the silent-no-cache gap for users routing Anthropic/Bedrock through an openai-compatible proxy (Bifrost, LiteLLM). The pre-patch caching path either set `promptCacheKey` (an OpenAI-native option proxies don't translate) or matched on `model.id.includes("claude")` and ran the Anthropic-style `applyCaching`, which still set message-level fields the proxy strips. New `applyCompatCaching` in `provider/transform.ts:417-461` instead writes Anthropic-style `cache_control: { type: "ephemeral" }` into a per-block `providerOptions.openaiCompatible` envelope — what both proxies actually forward upstream.

## Specific references

- `config/provider.ts:90-94`: new `cacheStrategy: Schema.Literals(["bedrock"])` provider option.
- `provider/transform.ts:421-430`: target selection — first 2 system messages + last 2 non-system messages, deduped via `unique([...])`.
- `provider/transform.ts:432-440`: string `system` content gets converted to `[{ type: "text", text, providerOptions: cacheOpt }]` (the cast `as unknown as ModelMessage` flagged in the comment).
- `provider/transform.ts:442-454`: user messages — string → `[{type:"text"}]`, array stays array, last block gets `mergeDeep`'d `providerOptions`.
- `provider/transform.ts:471-477`: `applyCaching` is now gated off for `@ai-sdk/openai-compatible` so the two paths don't double-fire.
- `provider/transform.ts:485-491`: trigger condition — `cacheStrategy === "bedrock"` OR (`setCacheKey === true` AND `model.id.toLowerCase().includes("bedrock/")`).
- `session/llm.ts:399`: now passes `item.options` as 4th arg `providerOpts` so config-level provider options reach the transform.
- `test/provider/transform.test.ts:120-235+`: new describe block — three tests covering string→array conversion, array passthrough with last-block annotation, and the `setCacheKey + bedrock/` heuristic path.

## Concerns / nits

- **Cast at `:438`** (`as unknown as ModelMessage`) is load-bearing — the @ai-sdk types disallow array content for `system`. Worth a brief comment that proxies serialize the array correctly even though the type rejects it (the inline comment is good but could call out the runtime risk if the type ever tightens).
- The bedrock/ heuristic at `:489` is case-insensitive on the model id but the json schema literal `"bedrock"` for `cacheStrategy` is not — fine, but if another backend (e.g. `vertex/`) needs the same treatment, the literal type will need to widen alongside the heuristic.
- `applyCompatCaching` only annotates user/system; the existing `applyCaching` also tags assistant messages for some providers. Confirm proxies don't need cache_control on assistant turns for prompt-cache hit accounting (Anthropic native supports it).
- `unique([...])` import not visible in the diff snippet — confirm it's the standard helper used elsewhere in the file (vs e.g. `[...new Set(...)]` which would lose object identity).
- Docs section (`packages/web/src/content/docs/config.mdx`) is touched (+31) but not in the visible 200-line diff; assume it documents `cacheStrategy: "bedrock"` use.

## Verdict

**merge-after-nits (man)** — solid bug fix with focused tests; small documentation/comment polish would harden the contract.
