# sst/opencode PR #25357 — feat(provider): add `preserveReasoningInContent` config option

- **Head SHA**: `aabf75279abe39fa2cadd2247c7fdc0e2823f0b3`
- **Scope**: schema + transform.ts (Qwen3.6 `preserve_thinking` interop)

## Summary

Adds an opt-in `preserveReasoningInContent` model option. When set, `normalizeMessages` in `packages/opencode/src/provider/transform.ts:217-246` rewrites assistant messages by:

1. extracting `reasoning` parts,
2. concatenating them inside `<thinking>...</thinking>`,
3. prepending the result as a `text` part,
4. clearing `providerOptions.openaiCompatible.reasoning_content`.

Schema gains a typed field on `Model.options` via `Schema.StructWithRest` at `packages/opencode/src/config/provider.ts:56-63`, preserving the open record so existing extra options keep working.

## Comments

- Logic at line 218 ORs an `_options` arg with `model.options` — good, request-level wins over model-level.
- `(model.options as any)` cast at line 220 is justified because the schema fans out via `StructWithRest`; consider extracting a typed accessor later but not blocking.
- The new prepended text part has a hard `\n\n` separator (line 230). Safe; downstream tokenizers won't choke.
- Setting `reasoning_content: undefined` (line 237) inside a spread — relies on the provider serializer dropping `undefined`. If any downstream JSON serializer keeps `undefined`-as-null, this becomes a regression. Worth a unit test pinning the outgoing payload shape for the openai-compatible provider.
- No tests added in the diff for the transform branch itself — the schema test alone is insufficient for a routing change like this.

## Verdict

`merge-after-nits` — add at least one test asserting the rewritten message shape and the absence of `reasoning_content` for the openai-compatible path.
