# sst/opencode #24734 — fix(XXX): ensure available variants sync from api

> Note: redacted product name in title per repo conventions.

- PR: https://github.com/sst/opencode/pull/24734
- Head SHA: `c4481d9be337179fdbc6ae31e226facd98d7f8a0`
- Files changed: 1 (`+3/-1`) — `packages/opencode/src/provider/provider.ts`
- No linked issue; PR body is empty.

## Verdict: needs-discussion

## Rationale

- **Surgical 2-line conditional flip in a hot path with no body, no linked issue, no test.** The change at `provider.ts:1361-1363` wraps the existing `model.variants = mapValues(ProviderTransform.variants(model), (v) => v)` assignment in `if (!model.variants || Object.keys(model.variants).length === 0)`. Stated intent (from title): when the API already returned `model.variants`, don't overwrite it with whatever `ProviderTransform.variants()` derives. When the API didn't, fall back to the derived value as before.
- **The shape of the fix is plausible but the regression risk is non-trivial.** `ProviderTransform.variants(model)` is the canonical normalizer that the rest of the layer code (`:1364-1370`, the immediately following `configVariants` merge) assumes has been applied. By short-circuiting it whenever the upstream API returns *any* variants object — even one with one entry — the PR is implicitly trusting that the API's variant shape is byte-compatible with whatever `ProviderTransform.variants` would have produced. If the API returns variants with a different normalization (e.g. missing a derived `default` key, different price unit conventions, missing `cost.input`/`cost.output` shape), every code path downstream that previously assumed normalized shape now sees the raw API shape on this provider only.
- **The `configVariants` merge at `:1364-1370` (next 7 lines, unchanged by this diff) does `Object.assign(model.variants, configVariants)` to layer user config on top.** That merge implicitly assumes `model.variants` already has the canonical shape. If the API-returned variants are missing fields the user's config also doesn't override, the merged result is now *partially-shaped* in production for the first time. Without a test, there's no way to tell from the diff whether the provider that triggered this fix returns variants that pass through `ProviderTransform.variants` as a no-op.

## Blockers

- **No test.** This is a 2-line change to a load-bearing path (`provider/provider.ts` runs on every refresh). At minimum need a unit test pinning two cells: (1) API returns `variants: {}` → derived variants are populated as before; (2) API returns `variants: { foo: {...} }` → user-supplied entry survives, derived isn't applied, and the downstream `configVariants` merge still produces a usable result.
- **No PR body explaining the failure mode.** Title says "ensure available variants sync from api" but doesn't name the bug symptom, the affected provider, or what the user sees. From the diff alone a reviewer cannot tell whether this is a "API returns variants that get clobbered" fix (the obvious read) or a "ProviderTransform.variants returns nothing because model shape changed" fix (which would have a different correct fix).
- **The empty-object check `Object.keys(model.variants).length === 0` is a JS-engine perf footgun on hot paths.** `Object.keys` allocates an array — fine for a sync-once code path, worth flagging as `for (const _k in model.variants) { has = true; break }` if this runs per refresh per model. Probably negligible at expected fan-out, but worth confirming with the author since the change has no perf-justification context.

## What "needs-discussion" means here

This is plausibly correct but ships with the minimum-viable diff and zero verification artifact for a behavior change in the variants pipeline that has multi-PR-of-history shape-fragility (cf. drip-129 / drip-134 nearby `tool_choice`-translation and provider-variant work). Asking the author to (a) name the failing scenario in the body, (b) add a unit test pinning both cells, and (c) confirm the affected provider's API shape is byte-compatible with `ProviderTransform.variants` output is the right gate.
