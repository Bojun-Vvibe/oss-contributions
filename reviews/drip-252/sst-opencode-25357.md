# sst/opencode #25357 — feat(provider): add preserveReasoningInContent for Qwen preserve_thinking

- **Repo:** sst/opencode
- **PR:** https://github.com/sst/opencode/pull/25357
- **HEAD SHA:** `aabf75279abe39fa2cadd2247c7fdc0e2823f0b3`
- **Author:** jgrcic
- **Verdict:** `merge-after-nits`

## What the diff does

Two-file change adding an opt-in `preserveReasoningInContent: true` flag to
the Model schema for OpenAI-compatible providers (Qwen3.6 over local vLLM is
the named target):

1. `packages/opencode/src/config/provider.ts:53-65` — replaces the open
   `Schema.Record(Schema.String, Schema.Any)` `options` slot with a
   `Schema.StructWithRest({preserveReasoningInContent: optional Boolean}, [Record(String, Any)])`
   so the new flag gets an explicit type + description annotation while
   the rest of `options` remains free-form pass-through.
2. `packages/opencode/src/provider/transform.ts:217-247` — when the flag
   is true, walks each assistant message and, if any part has
   `type === "reasoning"`, joins all reasoning text and prepends a single
   `{type: "text", text: "<thinking>${reasoningText}</thinking>\n\n"}`
   before the surviving non-reasoning parts, while also forcibly setting
   `providerOptions.openaiCompatible.reasoning_content = undefined` so
   the AI SDK doesn't *also* emit it natively in the OpenAI-compatible
   `reasoning_content` field.

## Why the change is right

The mechanism the author identifies is real: vLLM's `preserve_thinking`
feature only re-injects historical chain-of-thought into the model's
context if it's present *inside* the assistant `content` string wrapped
in `<thinking>...</thinking>` tags — the OpenAI-spec `reasoning_content`
sibling field is ignored for this code path on Qwen3.6. So the default
OpenCode shape (extract reasoning out into a sibling field) silently
breaks multi-turn coherence for any vLLM-Qwen user who enables
`preserve_thinking`.

Putting the flag at the Model schema rather than a global Provider knob
is the right granularity — the same vLLM endpoint can host multiple
models, only some of which want this transformation, and the Model-level
override is already the configuration unit that survives variant
overrides.

The asymmetric "set `reasoning_content: undefined` on `providerOptions`"
at `transform.ts:235-241` is the load-bearing detail — without it the
SDK would emit *both* the in-content `<thinking>` block *and* the
sibling `reasoning_content` field, doubling the historical reasoning
context (and on a small model, eating the context window twice).

## Nits (non-blocking)

1. **Zero direct unit tests.** `transform.test.ts` has no entry exercising
   `preserveReasoningInContent: true` — the load-bearing
   `<thinking>${reasoningText}</thinking>\n\n` shape, the
   `providerOptions.openaiCompatible.reasoning_content = undefined`
   override, and the "no-op when flag absent" path are all unverified.
   At minimum: one test asserting an assistant message with
   `[{type: "reasoning", text: "x"}, {type: "text", text: "y"}]`
   becomes `[{type: "text", text: "<thinking>x</thinking>\n\n"}, {type: "text", text: "y"}]`
   with `reasoning_content` cleared.

2. **`(_options as any)?.preserveReasoningInContent`** at `:215-217` —
   `_options` is the per-call options bag and `model.options` is the
   config-time bag, both checked, but the `as any` cast bypasses the
   schema improvement made in change #1. If the typed schema work is
   the point, the call site should consume the typed shape.

3. **Empty-message safety not handled.** Unlike the sibling Bedrock
   normalize arm at `transform.ts:217-258` (drip-224 #25197) which
   inserts a `{type: "text", text: "..."}` placeholder when an
   assistant turn becomes empty after filtering, this arm does
   `[textPart, ...filteredContent]` — so the case of "assistant
   message that was *only* `[{type: "reasoning"}]`" (no surviving
   text) still produces a one-element content array carrying just
   the `<thinking>` block. That's probably fine for vLLM (the
   `<thinking>` tag *is* content from its perspective) but worth
   one sentence in a code comment locking the decision rather than
   leaving the reader to wonder if the empty-text branch needs a
   placeholder like Bedrock does.

4. **`reasoningText` join with no separator** at `:222`. If a turn
   has multiple reasoning parts (some providers emit reasoning
   chunks rather than one block), they get concatenated without a
   newline, making a glued blob inside `<thinking>`. `\n\n` between
   parts matches how the rest of the codebase joins reasoning
   stream chunks.

5. **Naming**: `preserveReasoningInContent` is accurate for the
   transformation but doesn't telegraph "you must set this for Qwen
   `preserve_thinking` over vLLM" — the doc-string at `provider.ts:57`
   does call out Qwen3.6 by name, which is the right place. Worth
   adding a one-line config-doc entry referencing this flag from the
   Qwen provider page so users hit it in the search path.

## Verdict rationale

Right-shaped surgical opt-in flag, gated correctly, with the
load-bearing dual-emission cleanup (`reasoning_content = undefined`)
in place. Ships with zero direct test coverage, which is the only
real blocker; the schema-vs-cast inconsistency, multi-part join
separator, and empty-content code comment are all polish nits.

`merge-after-nits`
