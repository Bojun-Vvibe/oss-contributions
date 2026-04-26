# QwenLM/qwen-code#3631 — Feat/stats model cost estimation

- **Repo**: QwenLM/qwen-code
- **PR**: [#3631](https://github.com/QwenLM/qwen-code/pull/3631)
- **Head SHA**: `f27d4dce5479`
- **Author**: B-A-M-N
- **Base**: `main`
- **Size**: +903 / −3, 10 files

## Context

Adds opt-in per-model cost estimation to the `/stats model` slash command and
to the `<ModelStatsDisplay />` TUI panel. Pricing is configured in
`settings.json` under a new `modelPricing` schema entry; cost is derived
from `inputPerMillionTokens` and `outputPerMillionTokens` against the
already-tracked `tokens.prompt` / `tokens.candidates` counters.

## Change

Five logical pieces:

1. **Settings schema** —
   `packages/cli/src/config/settingsSchema.ts:957-975` adds the
   `modelPricing` entry as `Record<string, {inputPerMillionTokens?,
   outputPerMillionTokens?}>` with `requiresRestart: false` and
   `showInDialog: false` (config-file only, deliberately not surfaced in
   the settings dialog).
2. **Cost calculator** —
   `packages/cli/src/utils/costCalculator.ts` (new, 33 lines) exports a
   pure `calculateCost({inputTokens, outputTokens, pricing}) → number |
   null` that returns `null` if `!pricing` or if both per-million values
   are missing/zero. The `total > 0 ? total : null` final clause is the
   "missing pricing means no row" gate consumed by the UI.
3. **Slash command path** — `statsCommand.ts:90-108` reads
   `context.services.settings.merged.modelPricing` and appends an
   `Estimated cost: $X.XXXX` line per model in non-interactive output.
4. **Interactive panel** — `ModelStatsDisplay.tsx:67-110, 226-248` uses
   `useSettings()` + a `hasPricing` flag (any model with non-null cost)
   to conditionally render a "Cost / Estimated" `StatRow` block at the
   bottom of the panel; per-model values are `$X.XXXX` or `'N/A'`.
5. **Tests** — `costCalculator.test.ts` (new, ~150 lines) covers
   missing-pricing → null, partial pricing (input-only or output-only),
   fractional rates (`expect(cost).toBeCloseTo(0.777777, 6)`), and the
   `toFixed(4)` UI rounding contract. `ModelStatsDisplay.test.tsx` and
   `statsCommand.test.ts` get a `SettingsContext.Provider` wrap and a
   "shows cost when pricing is configured" case.

## Strengths

- **Schema-first, opt-in design**: `modelPricing` defaults to `undefined`,
  the calculator returns `null` on missing config, the UI gates on
  `hasPricing`. Users with no pricing config see exactly the same UI as
  before — no regressions for the silent majority.
- **Pure calculator**: `calculateCost` is a 20-line pure function with
  exhaustive test coverage. Easy to reason about, easy to extend (e.g.
  for cache-read pricing tiers later).
- **Per-model resolution via `key.split('::')[0]`**: handles the
  `model::source` composite keys that `flattenModelsBySource` emits, so
  a `qwen3-coder` price entry covers both `qwen3-coder::main` and
  `qwen3-coder::tool` rows. Subtle but correct.
- **`toFixed(4)` decision is captured in a test**: the
  `expect(cost?.toFixed(4)).toBe('0.9839')` assertion locks the display
  precision so a future refactor can't silently widen it.

## Risks / nits

1. **Scope creep**: the diff also includes changes to
   `packages/core/src/models/modelConfigResolver.test.ts` adding
   `QWEN_CODE_API_TIMEOUT_MS` env-var override tests (visible at
   `modelConfigResolver.test.ts:260-340`). That's an unrelated feature
   under a `Feat/stats` PR title and should be split out. Either the
   author rebased two work-in-progress branches together, or the
   resolver fix is being smuggled in. Ask before merging.
2. **No cache-read pricing**: real-world bills have separate
   `cache_read` and `cache_write` rates that are 10–25% of normal input
   on most providers. The schema models only `inputPerMillionTokens` /
   `outputPerMillionTokens`, so a heavy-cache user will see a *higher*
   estimated cost than the actual bill. Ship it as v0 with a doc note
   that the estimate excludes cache-read discounts; defer the schema
   evolution to a follow-up.
3. **No reasoning-token line item**: same shape — reasoning tokens are
   billed at output rate by all providers I know of, but the metric
   isn't surfaced separately. The `tokens.candidates` counter likely
   already includes reasoning (mirrors the upstream OpenAI semantics
   reviewed in sst/opencode#24441), so the estimate may be correct by
   accident. Worth a comment on `costCalculator.ts:1` clarifying which
   counter feeds `outputTokens`.
4. **Currency assumption**: hardcoded `$` prefix in three places
   (`statsCommand.ts:106`, `ModelStatsDisplay.tsx:241, 244`). Trivial,
   but a `currency` field in `modelPricing` (default `"USD"`) would be a
   future-proof v1.
5. **`requiresRestart: false` claim**: the slash command reads
   `settings.merged.modelPricing` per invocation so this is true at
   read time. Confirm the settings reload path actually re-merges on
   file change (rather than caching at startup), otherwise the flag is
   misleading.

## Verdict

**needs-discussion** — the cost-estimation core is well-built and
test-covered, but the unrelated `QWEN_CODE_API_TIMEOUT_MS` resolver
changes need to come out first, and the v0 limitations
(cache-read accuracy, currency, reasoning-token framing) should be
acknowledged in a docs change before this lands so users don't file
"my bill doesn't match the estimate" issues.

## What I learned

Optional config that surfaces a *new column* in an existing TUI panel is
a good place to use a `hasPricing` aggregate gate rather than per-row
conditional rendering — keeps the layout stable and the no-config user's
experience unchanged. The `key.split('::')[0]` move is also a nice example
of why composite IDs should expose a `getDisplayName()` helper rather
than forcing every consumer to know the separator.
