# sst/opencode PR #24443 — preserve `reasoning_content` on second interleaved transform pass

- **PR**: https://github.com/sst/opencode/pull/24443
- **Author**: @claudianus (Ryan H. Park)
- **Head SHA**: `fa478297f13d87ae544034a07398d5a0bda7f336`

## Summary

Three changes in `packages/opencode/src/provider/{provider,transform}.ts`:

1. **Default `interleaved` capability for reasoning models.** In both
   `fromModelsDevModel` (`provider.ts:1009`) and the `layer.merge`
   path (`provider.ts:1180`), if `model.interleaved` is unset, fall
   back to `{ field: "reasoning_content" }` when the model is a
   reasoning model. Previously fell back to `false`.
2. **OpenRouter-routed DeepSeek detection** in `transform.ts:178-179`:
   the `if (model.api.id.includes("deepseek"))` check now also
   matches `model.id.includes("deepseek")`.
3. **The actual fix** in `transform.ts:196-225`: when the interleaved
   transform runs a second time (the message list re-flowing through
   `normalizeMessages`), the assistant-content `reasoning` parts have
   already been extracted on pass 1 and live only in
   `providerOptions[sdk][field]`. Pass 2 was overwriting that field
   with the empty string from `reasoningParts.map(...).join("")`,
   which DeepSeek then rejects with 400 *"reasoning_content must be
   passed back"*. Fix: read the pre-existing field value off
   `providerOptions` and prefer it when extraction yielded nothing.
4. **Bonus catch-all** at `transform.ts:232-265`: even when
   `interleaved` is *not* configured, if the model is a reasoning
   model, inject an empty-string `reasoning_content` on every
   assistant turn (covers historical messages stored before reasoning
   was enabled).

## Verdict: `merge-after-nits`

The core fix at `transform.ts:208-211` is correct and well-explained
in the inline comment. Three concerns before landing.

## Concerns

1. **`sdkKey` derivation moved inside the loop.** At `transform.ts:198`
   `const sdk = sdkKey(model.api.npm) ?? "openaiCompatible"` is hoisted
   above the map, good. But in the new bonus block at `:236-258` it's
   recomputed *inside* both `Array.isArray(msg.content)` and
   `typeof msg.content === "string"` branches. Hoist once at the top
   of that `if (model.capabilities.reasoning)` block.

2. **Bonus block changes the wire protocol for non-interleaved
   reasoning models.** Adding `reasoning_content: ""` to *every*
   assistant turn for any reasoning-capable provider is a much
   larger blast radius than the bug being fixed. Confirm against
   non-DeepSeek reasoning providers (Anthropic via OpenAI-compat,
   o1-mini through OpenAI Responses-compat shims, Qwen reasoning
   models) that an empty `reasoning_content` field is at worst a
   no-op and not 400'd. If unverified, gate the catch-all on
   `model.id.includes("deepseek") || model.api.id.includes("deepseek")`
   to scope it to the bug's actual cause.

3. **String → array coercion in the bonus block** at `:248`:
   `[{type:"text", text: msg.content}, {type:"reasoning", text:""}]`
   silently shape-shifts assistant string content to array form. Any
   downstream code that does `typeof msg.content === "string"` will
   stop matching. Verify storage/serialization layer is OK with this
   for historical-message replay.

## Specific references

- `transform.ts:208-211` — the actual bug fix. The comment
  ("@ai-sdk yG converter resolves reasoning_content from
  providerOptions → overwriting it with empty string on the second
  pass causes DeepSeek 400") is exactly what reviewers need.
- `provider.ts:1009,1180` — capability default flip. Reasonable.
- `transform.ts:179` — OpenRouter `model.id` fallback. Sensible.

## Nits

- A regression test that runs `normalizeMessages` twice on the same
  message array and asserts `reasoning_content` survives would lock
  this in permanently.
