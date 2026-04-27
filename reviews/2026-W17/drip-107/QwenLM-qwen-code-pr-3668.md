# QwenLM/qwen-code #3668 — feat(stats): add current session billing estimates

- **Repo**: QwenLM/qwen-code
- **PR**: #3668
- **Author**: shenyankm
- **Head SHA**: 0aff9a440e25206dc9ea34d1970dbd4a0f0c5d04
- **Base**: main
- **Size**: +1768 / −25 across 24 files. Largest blocks:
  `packages/cli/src/ui/utils/modelBilling.ts` (+248) and its test
  (+290), `packages/cli/src/ui/utils/sessionBilling.ts` (+160) and
  test (+125), `packages/cli/src/ui/contexts/SessionContext.test.tsx`
  (+117), `packages/cli/src/ui/components/SessionSummaryDisplay.test.tsx`
  (+110), `packages/cli/src/ui/components/ModelStatsDisplay.tsx`
  (+99/-10), `packages/core/src/telemetry/uiTelemetry.ts` (+76/-2),
  `packages/cli/src/config/settingsSchema.ts` (+73), and a
  documentation block in `docs/users/configuration/settings.md` (+65).

## What it changes

Adds an opt-in, settings-driven cost estimator for `/stats model` and
the session summary panel. Three integration points:

1. **Settings schema** (`packages/cli/src/config/settingsSchema.ts:108-150`)
   introduces `ModelTokenDiscounts`, `ModelTokenPrice`, and a
   `billing` block (`currency`, `modelPrices`) at the same top level
   as `model`/`tools`/`mcp`. Prices are per-million-tokens; keys are
   either bare model IDs or `authType:model` for provider-specific
   pricing. The merge test at `settings.test.ts:357-413` confirms
   user and workspace `billing.modelPrices` maps merge entry-by-entry
   (workspace wins per key, currency from user is preserved).
2. **Pricing math**
   (`packages/cli/src/ui/utils/modelBilling.ts:1-248` plus
   `sessionBilling.ts:1-160`). Token buckets recognized:
   `prompt - cached` → uncached input, `cached` → cached input,
   `candidates` → output. Per-bucket optional `discounts.input`,
   `discounts.cachedInput`, `discounts.output` (default 1.0 each;
   `cachedInput` defaults to `input` when `cachedInput` price is
   absent). The `getModelBillingBreakdown` function picks the most
   specific key (`authType:model` over bare `model`) and aggregates
   per-`authType` sub-totals so a single model used under multiple
   auth modes shows separate lines.
3. **UI surface** (`ModelStatsDisplay.tsx:63-150` and
   `SessionSummaryDisplay.tsx:+36`). The display block is gated on
   `hasBilling` — if no `modelPrices` entry matches any model in the
   session, the entire "Billing" section is suppressed. `currency`
   default is `"USD"` and is used as a prefix; the test at
   `ModelStatsDisplay.test.tsx:357-405` exercises the cached-aware
   path end-to-end.

`uiTelemetry.ts:+76/-2` is the supporting plumbing: per-model token
breakdowns gain `cached`-aware accumulators so the existing
telemetry surface knows about cached input separately from the rest.

## Strengths

- The pricing math is gated behind explicit user configuration. The
  `hasBilling` boolean at `ModelStatsDisplay.tsx:91-93` short-circuits
  rendering when `billingBreakdowns` is all `undefined`, so a user
  who hasn't set `modelPrices` sees the existing /stats output
  unchanged. No "estimated $0.00" surprises, no hardcoded provider
  price tables that go stale.
- The `authType:model` key precedence at
  `modelBilling.ts` (specific-over-general) is exactly right for the
  realistic case where the same `gpt-4o` is served by both an OpenAI
  direct key and a proxy with a different per-token rate. The
  fallback to bare `model` keeps small configs simple.
- Per-bucket discount multipliers (`ModelTokenPrice.discounts.input`
  / `cachedInput` / `output`, settings schema `:108-122`) handle the
  realistic case where a provider charges full price for output but
  discounts cached input; without per-bucket multipliers users would
  have to encode discounts as separate `modelPrices` entries.
- The settings merge test at `settings.test.ts:357-413` pins the
  union-of-keys merge semantics — if user settings price `qwen3.5-plus`
  and workspace settings price `deepseek-v4-flash`, the merged config
  has both. This is the right semantics for shared workspace
  configs that want to add prices on top of a personal default.
- `ModelStatsDisplay.test.tsx:357-405` is end-to-end: 1M uncached +
  250k cached input + 500k output at `input=$5/M, cachedInput=$1/M,
  output=$15/M` should yield $3.75 + $0.25 + $7.50 = $11.50, and the
  test asserts exactly those values appear in the rendered frame.
  That's the right level of test for a money-display feature where
  off-by-one or off-by-multiplier errors silently mislead users.
- Documentation at `docs/users/configuration/settings.md:211-280`
  explicitly says "estimates may differ from the final bill shown
  by your provider", which is the only honest disclaimer to make.

## Risks / nits

- The currency string is rendered as a prefix (`$3.75`) hardcoded in
  test expectations (`ModelStatsDisplay.test.tsx:401-404`). That
  matches `currency: "USD"` but doesn't generalize: `JPY` shouldn't
  be `$`, `EUR` should be `€` (or postfix), etc. The doc table at
  `settings.md:225` says "currency code used when displaying", but
  the test pins `$`. Worth either documenting that the prefix is
  always `$` (and `currency` only changes a label elsewhere), or
  routing through `Intl.NumberFormat` with the currency code.
- `cachedInput` price-derivation logic ("defaults to `input` price
  when `cachedInput` price is absent" per `settings.md:228`) is
  user-friendly but can mislead. A user who configures
  `input: 5.0` and forgets `cachedInput` will be billed cached
  input at the same rate as uncached, which is *more* than reality
  in most cases. A safer default is to leave `cachedInput`
  unpriced (and warn in the UI that cached input price is
  unconfigured) rather than silently inheriting `input`. Worth a
  note in the docs at minimum.
- `ModelTokenPrice` and `ModelTokenDiscounts` use camelCase
  (`cachedInput`), which is consistent with the rest of the JSON
  settings schema, but the `authType:model` key uses a colon
  separator that requires careful escaping in JSON path queries.
  Consider documenting that bare model IDs must not contain `:` to
  avoid collisions.
- The discount multiplier semantics ("0.25 means 25% of the
  configured input price") at `settings.md:230-231` is a divisor in
  most billing systems but a multiplier here. The doc is explicit,
  but a user setting `discounts.input: 0.25` expecting "discount by
  25%" (i.e., pay 75%) will be surprised. Worth renaming to
  `multipliers` (since they aren't really discounts) or to
  `priceMultiplier` / `priceFraction`.
- The merge semantics for `billing.currency` when both user and
  workspace settings set it isn't directly tested in the new
  `settings.test.ts:357-413` (workspace doesn't set `currency`,
  only `modelPrices`). The implicit assumption (workspace
  overrides user, like other settings) should be pinned with a
  test.
- 1768 added lines across 24 files is a large footprint for a
  feature that's gated. Splitting into "schema + math" and
  "UI surface" PRs would shrink review burden, though merging as
  one isn't unreasonable given the tests cover both halves.
- The new `uiTelemetry.ts:+76` cached-aware accumulators are not
  reviewed here in detail — worth confirming that they preserve
  backward-compat with consumers that currently sum `prompt` as
  total input (i.e., that nobody downstream is now going to
  double-count cached tokens).

## Suggestions

- Switch currency formatting to `Intl.NumberFormat(locale, { style:
  "currency", currency: settings.billing.currency })` so the label
  matches the configured code, not always `$`.
- Rename `discounts` → `priceMultipliers` (or document with an
  example: `discounts.input: 0.75 means 'pay 75% of the listed
  price'`), to head off the semantic confusion.
- Either drop the implicit `cachedInput → input` fallback or render
  a small warning ("cached-input price falling back to uncached
  rate; configure `cachedInput` for accuracy") next to the
  Billing block when the fallback was used.
- Add a test for the `currency` merge precedence between user and
  workspace `billing` blocks.
- Consider splitting future, follow-on features (per-tool cost,
  per-MCP server cost, monthly spend rollups) into separate PRs to
  keep this surface auditable.

## Verdict

`merge-after-nits` — the design is opt-in, the math is well-tested,
and the integration is properly gated on configured prices. The
currency-formatting and `discounts`-vs-`multipliers` semantic
clarifications are worth resolving before merging because they
affect the user-visible output and are hard to change later
without breaking existing configs.
