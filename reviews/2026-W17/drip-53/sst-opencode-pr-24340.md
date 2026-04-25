# sst/opencode#24340 — fix(acp): expose variant config option
**SHA**: `47582750` · **Author**: segfaultmedaddy · **Size**: +47/-1 across 1 file

## What
Surfaces model "variants" (e.g. `high` / `max` effort levels — see PR
24350 for how those get populated) as a first-class ACP
`SessionConfigOption` of type `select`. Previously the ACP agent
exposed only `model` and `mode` selectors; now `variant` is a third
option, gated behind `variantOptions.length > 1` so single-variant
models don't get a useless dropdown. Setting the value on the
`session/setConfig` side also routes through the session manager and
properly normalises the sentinel `DEFAULT_VARIANT_VALUE` back to
`undefined`.

## Specific cite
- `packages/opencode/src/acp/agent.ts:1846-1864` is the build-side:
  it constructs `variantOptions` with display-name title-casing
  (`split(/[_-]/).map(...).join(" ")`), then only pushes the option
  when there's more than one. Reasonable UX guard.
- Lines 1346-1352 are the set-side: validates `params.value` against
  `availableVariants` and rejects unknown values via
  `RequestError.invalidParams` — good, no silent acceptance.
- `buildConfigOptions` signature change at line 1830-1834 adds
  `currentVariant` and `availableVariants` as required (non-optional)
  params. All four call sites are updated in the diff (lines 663-672,
  1281-1287, 1366-1372). I'd double-check there isn't a stray helper
  call elsewhere in `acp/` that was missed.
- The `category: "effort"` string at line 1862 is a new category value;
  worth confirming the consuming UI (Zed's ACP client, etc.) already
  knows how to render it, or whether it should fall back to a generic
  category for older clients.

## Verdict: merge-after-nits
Clean, focused change with consistent updates at all call sites.
Two nits: (1) confirm `category: "effort"` is recognised by current
ACP consumers or document it in the schema, and (2) consider an ACP
contract test that asserts the dropdown disappears when a model has
exactly one variant — the boundary condition is the bug-prone one.
