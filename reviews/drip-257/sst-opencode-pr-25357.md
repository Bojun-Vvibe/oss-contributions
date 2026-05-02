# sst/opencode PR #25357 — feat(provider): add preserveReasoningInContent config option to fix Qwen preserve_thinking interoperability

- URL: https://github.com/sst/opencode/pull/25357
- Head SHA: `aabf75279abe39fa2cadd2247c7fdc0e2823f0b3`
- Author: jgrcic
- Verdict: **merge-after-nits**

## Summary

Adds a per-model option `preserveReasoningInContent` so that, when talking to providers (notably Qwen3.6 with `preserve_thinking`) that need historical reasoning re-injected as part of the assistant message *content* rather than as a separate `reasoning_content` field, the transformer rewrites prior assistant turns to wrap reasoning in `<thinking>...</thinking>` and strips the structured reasoning_content. The schema change makes the option a typed/known key while still allowing arbitrary additional `options.*` entries via `StructWithRest`.

## Line-level observations

- `packages/opencode/src/config/provider.ts` ~lines 53–65: `Model.options` migrates from a plain `Record<String, Any>` to `StructWithRest({ preserveReasoningInContent: optional Boolean }, [Record<String, Any>])`. This is a **schema-shape change**: anyone already validating with strict mode against the old `Record` could see the property surface as a typed field. The annotated description is good. Confirm there is no JSON-Schema export consumer (docs site / config generator) that needs to be regenerated alongside this PR — the diff does not appear to update any generated artifacts.
- `packages/opencode/src/provider/transform.ts` ~lines 214–246: the new block runs *after* the existing message normalization. The "or" between `_options?.preserveReasoningInContent` and `(model.options as any)?.preserveReasoningInContent` means call-site overrides take precedence and fall back to the model definition — that is the expected ordering.
- The implementation iterates `msg.content` twice (`filter` then `filter`). For very long histories this is two extra passes per assistant message. Minor; not worth blocking on.
- `reasoning_content: undefined` is being spread into `providerOptions.openaiCompatible`. Whether this actually *removes* the field downstream depends on how the provider serializes — `JSON.stringify` will drop `undefined` keys, but a typed builder may not. Worth a unit test to assert the wire payload no longer contains `reasoning_content`.
- The `<thinking>...</thinking>\n\n` text part is *prepended* to the filtered content. If the model previously emitted multiple separate `reasoning` parts interleaved with `text` parts, joining them all into one block at the front loses ordering information. For most use cases this is fine, but worth a comment noting the lossy collapse.

## Suggestions

1. Add a unit test in `provider/transform.test.ts` (or equivalent) covering: (a) option off → reasoning passes through unchanged; (b) option on → assistant history rewritten with `<thinking>` wrapper and `reasoning_content` absent from `providerOptions`.
2. Document the option in the provider docs (`packages/web/src/content/docs/.../providers.mdx` or similar) — schema annotation alone won't surface it for users browsing the site.
3. Drop the two `as any` casts on `(part: any)` by importing the part union type from the same module that defines `msg.content`'s element type.
4. Consider an `else` branch (or comment) clarifying behavior when `reasoningParts` is non-empty but `reasoningText` is empty (all-empty `text` fields) — current code falls through silently, which is probably correct but is worth pinning down.
