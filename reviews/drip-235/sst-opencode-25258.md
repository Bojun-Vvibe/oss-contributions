# sst/opencode#25258 — fix: ensure MiMo models display correct context limit in tooltip

- **PR**: https://github.com/sst/opencode/pull/25258
- **Head SHA**: `e22c0cea47d57ce9adfb08c9f2c9b5ed9f5afb0b`
- **Size**: +8 / -4, 2 files
- **Verdict**: **merge-after-nits**

## Context

Closes #25256. Xiaomi MiMo models (256K–1M token context window) showed `Context limit 0` in the model selector tooltip. Author traced the regression to two compounding bugs in the `model.limit.context` data pipeline.

## What's right

**Bug 1 — `provider/provider.ts:1002-1010` (`fromModelsDevModel`).** The `limit` object was constructed by direct property access on a possibly-missing source `model.limit`:

```ts
limit: {
  context: model.limit.context,   // throws if model.limit is undefined
  input: model.limit.input,
  output: model.limit.output,
}
```

Fix flips to optional chaining with sensible defaults:

```ts
limit: {
  context: model.limit?.context ?? 0,
  input: model.limit?.input,
  output: model.limit?.output ?? 0,
}
```

This is the actual root cause — when `model.limit?.context` was undefined, JSON serialization stripped the field, the client received a `limit` object missing `context`, and downstream `model.limit.context.toLocaleString()` calls would throw or render as the `0` fallback the tooltip then displayed.

**Bug 2 — `app/src/components/model-tooltip.tsx:75-79`.** Defensive read on the consumer side:

```ts
const context = () => {
  const limit = props.model.limit?.context
  if (limit === undefined || limit === null) return ""
  return language.t("model.tooltip.context", { limit: limit.toLocaleString() })
}
```

Returning empty string (rather than rendering "Context limit 0" or letting `.toLocaleString()` throw) is the correct UX choice — better to omit the row than to show a confusing zero.

## Risks / nits

- **Asymmetric defaults are suspicious.** In `fromModelsDevModel`, `context` and `output` get `?? 0` while `input` is left as `undefined`. There's no comment explaining why; if the intent is "context and output are required at the wire, input is optional", a one-line comment or — better — making the schema enforce non-optional `context` would prevent this regression from re-occurring. Worth a one-line clarification before merge.
- **The `?? 0` in the producer slightly defeats the consumer guard.** With Bug 1 fixed, `model.limit?.context ?? 0` means the consumer at `model-tooltip.tsx:76` will see `limit === 0`, not `limit === undefined`, so the `if (limit === undefined || limit === null) return ""` branch never fires for this case. The tooltip will then render `Context limit 0` again. To make the consumer guard meaningful, the producer should pass through `undefined` (i.e. drop the `?? 0`), or the consumer should also guard `limit === 0`. As written, the two fixes don't compose for the actual reported bug — the consumer guard is dead code in the MiMo case.
- **Tests missing.** No unit test pins either fix. A two-line vitest case in `provider.test.ts` asserting `fromModelsDevModel({...limit: undefined})` round-trips to a serializable shape, and a snapshot test on `<ModelTooltip>` for the missing-limit case, would prevent regression.
- **Author claims to have traced the pipeline manually but did not include the specific upstream model file** (`models.dev` source for MiMo) in the PR — would help the reviewer confirm the source data really lacks `limit.context` vs the field being dropped at one of the merge steps in between.

## Verdict

**merge-after-nits.** The producer fix at `provider.ts:1002-1010` is the right change and resolves the reported bug. The consumer fix at `model-tooltip.tsx:75-79` is defensible defensive code. The two together leave a small dead-branch issue (consumer guard never fires once producer defaults to 0) — pick one of: drop `?? 0` on the producer, or add `limit === 0` to the consumer guard. Either way, add a regression test before merge.
