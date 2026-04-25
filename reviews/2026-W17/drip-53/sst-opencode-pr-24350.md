# sst/opencode#24350 — fix: deepseek when using messages api
**SHA**: `5338bc9d` · **Author**: rekram1-node · **Size**: +123/-0 across 4 files

## What
Adds `deepseek` as a recognized model id in the variant-builder for the
Anthropic Messages API path, mapping it to two effort levels (`high`,
`max`) that emit both a `thinking: { type: "enabled" }` block and an
`output_config.effort` field. To make the latter actually reach the
wire, a vendored patch is added against `@ai-sdk/anthropic@3.0.71`
adding an `output_config` passthrough to `anthropicLanguageModelOptions`,
plus the `Object.keys(...).length > 0` gating change so the
`output_config` envelope is emitted whenever a passthrough has any
keys (not just when `effort` / `taskBudget` / structured-output are set).

## Specific cite
- `packages/opencode/src/provider/transform.ts:613-633` adds the
  `model.api.id.includes("deepseek")` branch — note it lands above the
  generic `adaptiveEfforts` branch, so for any provider that exposes
  deepseek via a non-standard model id this short-circuits before the
  shared logic. Worth confirming `includes()` is intentional vs an
  exact id check (could collide with custom aliases like `deepseek-r2`
  in unrelated providers).
- `patches/@ai-sdk%2Fanthropic@3.0.71.patch` lines 11-14 (in `dist/index.js`)
  introduce the schema field; lines ~3070-3080 change the conditional
  from `(effort || taskBudget || structuredOutput)` to additionally
  trigger on `Object.keys(output_config).length > 0`. This is a real
  semantic widening — previously, passing only `output_config` with no
  `effort` would have been silently dropped.
- The patch touches all four bundle variants (`dist/index.js`,
  `dist/index.mjs`, `dist/internal/index.js`, `dist/internal/index.mjs`)
  and both `.d.ts` files — that's the right thing to do for a bun
  patched dependency, but it's also a heavy maintenance burden the next
  time `@ai-sdk/anthropic` bumps.

## Verdict: merge-after-nits
Functionality is sound and the patch is correctly applied across all
bundle entry points. Two nits before merge: (1) tighten the deepseek
match to an exact id or prefix to avoid collisions, and (2) file an
upstream issue against `ai-sdk/anthropic` for the `output_config`
passthrough so this patch can be dropped on the next dep bump.
