# anomalyco/opencode PR #24218 — fix(provider): auto-enable interleaved for reasoning models

- **Repo:** anomalyco/opencode
- **PR:** [#24218](https://github.com/anomalyco/opencode/pull/24218)
- **Head SHA:** `2750b3a0bd196b515d853e310410398551332a63`
- **Author:** fkyah3
- **Size:** +1 / −1 across 1 file (one-liner)
- **Reviewer:** Bojun (drip-26)

## Summary

A genuine one-line fix. When a model is registered against the
`@ai-sdk/openai-compatible` provider with `reasoning: true` but no
explicit `interleaved` capability, the previous fallback chain
landed on `false`, and `reasoning_content` in `providerOptions`
was silently dropped during message replay. The fix tacks one
more fallback rung onto the existing `??` chain in
`packages/opencode/src/provider/provider.ts:1180`: if `reasoning`
is set, default `interleaved` to `{ field: "reasoning_content" }`
instead of `false`.

## The change

### `packages/opencode/src/provider/provider.ts:1180`

```ts
// before:
interleaved: model.interleaved ?? existingModel?.capabilities.interleaved ?? false,

// after:
interleaved: model.interleaved ?? existingModel?.capabilities.interleaved ?? (model.reasoning ? { field: "reasoning_content" } : false),
```

Precedence is preserved exactly:

1. explicit `model.interleaved` always wins;
2. `existingModel?.capabilities.interleaved` (e.g. data sourced
   from models.dev) wins next;
3. *(new)* if `model.reasoning` is truthy, default to
   `{ field: "reasoning_content" }`;
4. otherwise `false` (non-reasoning model regression-safe).

So the change is strictly a refinement of the *terminal* fallback,
never overriding upstream sources.

## What's good

- Correct precedence: this only changes behaviour when both
  `model.interleaved` and `existingModel?.capabilities.interleaved`
  are unset *and* `model.reasoning` is truthy. No model that
  previously worked can stop working.
- `{ field: "reasoning_content" }` matches the convention the rest
  of the openai-compatible adapter already uses for reasoning
  payload extraction, so no downstream parser change is needed.
- The PR body identifies the right symptom ("reasoning_content in
  providerOptions silently dropped during message replay") and
  the right test cases (reasoning-true default, non-reasoning
  default, explicit override, models.dev-sourced interleaved).
- Closes #24104 (related), keeping the linkage discoverable.

## Concerns

1. **No unit test added.** This is the most common failure mode
   for one-liner provider-config fixes — the PR body says "Tested
   locally" but there's no automated regression guard. A test in
   `packages/opencode/test/provider/` covering the four cases the
   PR body claims (reasoning-default, non-reasoning-default,
   explicit-override, models.dev-source) would be ~30 lines and
   would prevent a future precedence reorder from re-introducing
   the bug. **Should be a `merge-after-nits` blocker for any
   provider config path** — this file is high-traffic and the
   fallback chain is exactly the kind of code that gets reordered
   "just for readability" later.

2. **Default field name is hard-coded.** `"reasoning_content"` is
   the de-facto OpenAI-compatible name (DeepSeek, Together,
   Fireworks, vLLM with reasoning patches all use it). But some
   compatible providers emit `reasoning` as the field name
   instead. If a new provider hits opencode with
   `reasoning: true` but emits `reasoning` (no `_content` suffix),
   this fix will silently drop their content the same way the
   original bug did. Worth either:
   - documenting that providers wanting a different field must
     set `model.interleaved` explicitly; or
   - making the default field name configurable at the provider
     level (e.g. `provider.reasoningField ?? "reasoning_content"`).

3. **Long ternary on a long line.** The replacement line is
   ~135 characters wide and packs four levels of fallback into a
   single expression. A small refactor:

   ```ts
   const defaultInterleaved = model.reasoning
       ? { field: "reasoning_content" }
       : false
   const interleaved =
       model.interleaved
       ?? existingModel?.capabilities.interleaved
       ?? defaultInterleaved
   ```

   …would make the precedence visually obvious and survive
   future edits better. Pure readability, no behavioural change.

## Risk

Very low *for behaviour*. Strictly an additive fallback rung; no
existing model configuration path changes.

Slightly elevated *for regression*, because there's no test. If
someone later normalises the chain into a `defaultInterleaved`
helper without thinking about the new branch, the fix can quietly
revert.

## Verdict

**merge-after-nits**

The fix itself is correct and minimal. Block on adding the four
unit cases the PR body already enumerates manually — they're free
to write and they protect against the most likely regression
mode (precedence reorder in a future "cleanup" PR).

## What I learned

Provider capability fallback chains that end in a literal `false`
are a smell — they mean "we silently lost a feature for any model
that didn't explicitly opt in". When the upstream feature
(`reasoning`) is itself a boolean already on the model record, the
right pattern is to derive the dependent capability from it
*conditionally*, not to leave both as independent flags that the
user has to remember to set in pairs.
