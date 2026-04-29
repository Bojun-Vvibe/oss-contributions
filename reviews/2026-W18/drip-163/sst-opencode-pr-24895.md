# sst/opencode#24895 — fix: generalize reasoning_content injection across AI SDK providers

- URL: https://github.com/sst/opencode/pull/24895
- Head SHA: `c42e10571ee36956d257dae1fc00ba416fef7e35`
- Size: +62 / −35, 1 file (`packages/opencode/src/provider/transform.ts`)
- Verdict: **merge-after-nits**

## Summary

Replaces the hardcoded DeepSeek-only reasoning-content injection branch with a generic capability-driven pass. Adds `providerOptionsKey(model)` that resolves to `"openaiCompatible"` for both `@ai-sdk/openai-compatible` and `@openrouter/ai-sdk-provider` (because `getOpenAIMetadata()` upstream hardcodes that key regardless of wrapping provider name). Detects "needs reasoning" via `isInterleaved || model.capabilities.reasoning || msgs.some(...has reasoning part)` and injects empty-string `reasoning_content` for both array-content and string-content (legacy DB) assistant messages. Also unifies four `id.includes("deepseek-xxx")` temperature-override checks into a single `id.includes("deepseek")`.

## Specific issues flagged

### 1. `providerOptionsKey()` fallback to `"openaiCompatible"` for *every* unknown npm is too eager

At `transform.ts:50-58` the helper:

```ts
function providerOptionsKey(model: Provider.Model): string {
  const npm = model.api.npm
  if (npm === "@ai-sdk/openai-compatible" || npm === "@openrouter/ai-sdk-provider") {
    return "openaiCompatible"
  }
  return sdkKey(npm) ?? "openaiCompatible"  // <-- fallback
}
```

If `sdkKey(npm)` returns `undefined` (i.e., we don't recognize the provider at all), this defaults to `"openaiCompatible"`. That means a user who plugs in, say, a custom Anthropic-shaped wrapper that *isn't* OpenAI-compatible will get `providerOptions: { openaiCompatible: { reasoning_content: "" } }` injected — at best ignored, at worst rejected by a strict provider that validates `providerOptions` keys. Suggest returning `undefined` and short-circuiting the `hasReasoningContent` branch (no injection) when we don't know the key, rather than guessing.

### 2. `hasReasoningContent` short-circuits on "any message in history has a reasoning part"

At `transform.ts:217-225` (per the diff context around the new conditional):

```ts
const hasReasoningContent =
  isInterleaved ||
  model.capabilities.reasoning ||
  msgs.some((msg) => msg.role === "assistant" && Array.isArray(msg.content) &&
                     msg.content.some((part: any) => part.type === "reasoning"))
```

This is correct for "DB-replayed conversation came from a reasoning model so we must keep injecting". But note the **third leg flips behavior mid-conversation**: if a user starts with a non-reasoning model and switches mid-session to a reasoning one, the first reasoning-message arrival flips this `true` and *retroactively* coats every prior assistant message with `reasoning_content: ""`. Probably what we want, but worth an inline comment because future debuggers tracing "why is `reasoning_content: ""` on a 6-month-old message" will land here without context.

### 3. String-content assistant message branch widens injection scope without a capability check

At the new branch (per diff context "Also handle string-content assistant messages (old DB format)"):

```ts
if (msg.role === "assistant" && typeof msg.content === "string") {
  return {
    ...msg,
    providerOptions: {
      ...msg.providerOptions,
      [key]: { ...msg.providerOptions?.[key], [field]: "" },
    },
  }
}
```

This only fires inside the `if (hasReasoningContent)` block, so it's gated. Good. But the old code did `[...(msg.content ? [{type:"text",text:msg.content}] : []), {type:"reasoning",text:""}]` — i.e., it *also* converted the string to an array and appended a reasoning part. The new code keeps `content` as a string and only sets `providerOptions`. Confirm with a DeepSeek round-trip integration test (not just unit) that the API accepts `content: "string"` + `providerOptions.openaiCompatible.reasoning_content: ""` — the old code-path's "convert to array" suggests there may have been a real reason the AI SDK shape required array-form.

### 4. Temperature-override consolidation `deepseek-chat|reasoner|r1|v3 → deepseek` is a substring widen

At `transform.ts:476-478`:

```ts
if (
  id.includes("deepseek") ||
  id.includes("minimax") || ...
```

Previously matched `deepseek-chat`, `deepseek-reasoner`, `deepseek-r1`, `deepseek-v3`. Now matches *anything* with `"deepseek"` in the id — including hypothetical future `deepseek-coder`, `deepseek-vl`, third-party `my-deepseek-finetune-v1`, etc. Probably the desired behavior (any DeepSeek-derived model wants the same temperature variants) but call out in the PR body and add a smoke test that asserts `id="deepseek-coder-v2"` produces the variants.

### 5. `reasoningText || ""` may unintentionally swallow `0` or other falsy

At the new line (per diff context `[field]: reasoningText || ""`): `reasoningText` is built from `reasoningParts.map(...).join(...)`. If for any reason a reasoning part's text is `"0"` or numeric zero coerced to string, `||` works fine here (string `"0"` is truthy). But the change from the old `[field]: reasoningText` (no fallback) to `[field]: reasoningText || ""` means the empty-array case now gets `""` instead of `""` (since `[].join("")` is already `""`). The `|| ""` is dead code. Either remove it or document why it's defensive.

## Why merge-after-nits

Real fix to a real bug (DeepSeek-via-OpenRouter was getting wrong `providerOptions` key). Items 1-3 are correctness on the edges, 4-5 are clarity. None block merge given DeepSeek/OpenRouter are the only consumers in practice today.
