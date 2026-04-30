# sst/opencode#25110 — fix(opencode): ensure DeepSeek reasoning_content round-trips for all interleaved cases

- PR: https://github.com/sst/opencode/pull/25110
- Head SHA: `2ac26a3615...`
- Author: nahoskins
- Files: 2 changed, ~+230 (mostly tests)

## Context

DeepSeek's chat-completions API requires the previous turn's `reasoning_content` (even when empty `""`) to be echoed back on every assistant message in subsequent requests, especially across tool-call turns. opencode's `provider/transform.ts:normalizeMessages` only extracted reasoning into `providerOptions.openaiCompatible.reasoning_content` when `model.capabilities.interleaved` was the `{ field: "reasoning_content" }` object shape. DeepSeek model definitions from models.dev sometimes ship with `interleaved: true` (boolean) or no `interleaved` field at all, in which case reasoning was left in `content` and never moved to providerOptions, causing the next turn to fail/regress.

## Design

The conditional at `transform.ts:214-225` is widened from "interleaved object with field" to a disjunction:

```ts
const shouldInterleave =
  (typeof model.capabilities.interleaved === "object" &&
    model.capabilities.interleaved.field &&
    model.api.npm !== "@openrouter/ai-sdk-provider") ||
  (model.api.npm === "@ai-sdk/openai-compatible" && model.api.id.toLowerCase().includes("deepseek"))
```

The field name is then resolved with a default of `"reasoning_content"` when the capabilities object doesn't specify one. Net effect: any model on `@ai-sdk/openai-compatible` whose `api.id` contains "deepseek" gets reasoning extracted into providerOptions regardless of how `interleaved` is declared.

Test coverage at `test/provider/transform.test.ts:1124-1262` is the strongest part of this PR — four new tests:

1. **Multi-turn tool calls preserve reasoning_content on all assistant messages** — full conversation with two assistant turns (one carrying tool-call, one carrying final text), asserts `result[1].providerOptions.openaiCompatible.reasoning_content === "Let me check the weather API."` and `result[3]....reasoning_content === "The weather data shows sunny conditions."`. Exact bug shape.
2. **Empty reasoning_content preserved** — DeepSeek's known behavior of returning `reasoning: ""` with tool calls; asserts the empty string survives the transform (line 192: `expect(result[1].providerOptions?.openaiCompatible?.reasoning_content).toBe("")`).
3. **`interleaved: true` boolean shape** — covers the models.dev shape that motivated the fix.
4. **`interleaved: false` / missing** — edge case that the previous code path silently dropped.

## Risks / nits

- The detection key `model.api.id.toLowerCase().includes("deepseek")` is a substring match. It will catch `deepseek-chat`, `deepseek-reasoner`, `deepseek/deepseek-chat`, etc. — fine. It will also match a hypothetical fine-tune named `acme-deepseek-distill`, which is probably the right behavior here (same wire protocol). Not a concern.
- The OpenRouter exclusion only applies to the object branch, not the new DeepSeek branch — `model.api.npm === "@ai-sdk/openai-compatible"` makes that distinction by construction (OpenRouter uses a different npm package). Correct, but a one-line comment explaining "OpenRouter handles reasoning translation itself, that's why it's excluded; DeepSeek-via-openai-compatible needs us to do it explicitly" would future-proof the next reader.
- The hardcoded `"reasoning_content"` default field name is reasonable for now (DeepSeek is the only provider hitting this path), but if Qwen-style providers ever land on `@ai-sdk/openai-compatible` with their own `reasoning_content`-equivalent under a different name, this default will silently misroute. Worth a TODO note for then.

## Verdict

**merge-after-nits** — the comment about OpenRouter exclusion + a brief inline rationale for the DeepSeek substring detection would close this out. Behavior and tests are correct.
