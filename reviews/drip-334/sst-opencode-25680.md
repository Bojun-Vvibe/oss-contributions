# sst/opencode#25680 — fix: propagate hashline tool.execute.before args + ref params

- **Head SHA:** `8cc7db039417c31a92699d8732f65c5ceffd1560`
- **Author:** community
- **Size:** +20 / −2, 1 file (`packages/opencode/src/session/prompt.ts`)
- **Closes:** #25674

## Summary

Two coupled fixes in `prompt.ts`:

1. Inject hashline ref-based edit parameters (`startRef`, `endRef`, `ref`, `operation`, `content`, `fileRev`, `expectedFileHash`, `safeReapply`, `operations`) into the `edit` tool's JSON Schema so models with strict schema validation (e.g. DeepSeek) can use them.
2. Make `tool.execute.before` plugin hook output (`{ args }`) actually flow into the subsequent `item.execute(args, ctx)` call, so plugin mutations of `args` take effect.

## Specific citations

- `prompt.ts:421-438` — schema-injection block guarded by `if (item.id === "edit" && schema?.properties)`. Adds 9 properties plus optional `oldString`/`newString` removal from `schema.required` (`:436-438`). Mutation is in place on the schema returned from `ProviderTransform.schema(...)` — relies on that returning a fresh object, which a quick check is warranted on (see nit below).
- `prompt.ts:445-449` — introduces `const hookOutput = { args }` and passes it as the third arg to `plugin.trigger("tool.execute.before", ..., hookOutput)`, then `item.execute(hookOutput.args, ctx)`. This is the actual behavior fix: previously a plugin's mutation of `hookOutput.args` was discarded because `item.execute(args, ctx)` re-read the original `args`.

## Verdict: `request-changes`

The intent is right, but two issues:

1. **Hardcoded schema-injection bypasses extensibility.** Hashline-specific properties are inlined into a generic `if (item.id === "edit")` branch in core `prompt.ts`. Any future tool with similar needs will require the same surgical edit. A cleaner shape: have the `edit` tool itself (or a `schemaExtensions` registry on the tool definition) own these properties so the prompt builder doesn't need to know about hashline. This also fixes the silent coupling where `prompt.ts` and the actual hashline edit implementation must agree on parameter names.
2. **Schema mutation may corrupt cache.** `schema.properties.content = {...}` and `schema.required = schema.required.filter(...)` mutate the object returned from `ProviderTransform.schema(input.model, EffectZod.toJsonSchema(item.parameters))`. If `EffectZod.toJsonSchema` or `ProviderTransform.schema` ever memoizes, subsequent calls will see the augmented schema. Recommend `const schema = structuredClone(...)` before the `if` block, or build a new `properties`/`required` rather than mutating in place.

Plugin-args propagation fix (`:445-449`) is correct on its own and could be split into its own PR.
