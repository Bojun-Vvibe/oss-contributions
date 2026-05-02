# Review: QwenLM/qwen-code#3780

- **PR:** QwenLM/qwen-code#3780
- **Head SHA:** `3aefb8bbc1e857bec332d77ec2aa986d7e3001cb`
- **Title:** Feat/stats model cost estimation rebase
- **Author:** B-A-M-N

## Files touched (high-level)

- `packages/cli/src/config/settingsSchema.ts` — adds a new top-level `modelPricing` settings entry: `Record<string, { inputPerMillionTokens?: number; outputPerMillionTokens?: number }> | undefined`, category Model, no restart required, hidden from the in-app settings dialog (`showInDialog: false`).
- `packages/cli/src/ui/commands/statsCommand.test.ts` — adds three Vitest cases pinning behavior of `/stats model`: shows estimated cost when pricing is configured, omits cost line when pricing is absent, computes per-model cost when multiple entries exist. Helper `toModelMetrics(core)` adapts `ModelMetricsCore` → `ModelMetrics` by injecting `bySource: { [MAIN_SOURCE]: core }` to match the new metrics shape.

## Specific observations

- `settingsSchema.ts:967-985`: `modelPricing` is correctly typed `as | Record<string, { inputPerMillionTokens?: number; outputPerMillionTokens?: number }> | undefined`. Both pricing fields are optional, which means a user can provide only `inputPerMillionTokens` without `outputPerMillionTokens` — the cost calculator needs to handle missing fields gracefully (skip the missing component, don't `NaN`-out the total). The diff doesn't show the calculator, but the test `'stats model shows cost when pricing is configured'` only exercises both-fields-set, so the partial-config path may be untested.
- `settingsSchema.ts:984`: `description` example uses `qwen3-coder` with pricing `{0.30, 1.20}` (USD per million tokens). Example pricing values look plausible. `showInDialog: false` is the right call — this is power-user config, putting it in the model picker dialog would clutter it.
- `statsCommand.test.ts:13-17`: `toModelMetrics` helper hardcodes `bySource: { [MAIN_SOURCE]: core }` — this assumes every model's metrics live under `MAIN_SOURCE`. If sub-agents / side-queries route to a different source key, the test fixtures won't reflect that, and the production cost calculator may double-count or miss spend depending on how it iterates. The test only validates the simple case.
- `statsCommand.test.ts:170-200` (`'shows cost when pricing is configured'`): asserts `'Estimated cost: $0.9000'` for `prompt=1_000_000, candidates=500_000` at `{0.30, 1.20}` per million → `1.0 * 0.30 + 0.5 * 1.20 = 0.30 + 0.60 = 0.90`. Math checks out. 4-decimal precision is reasonable for sub-dollar costs.
- `statsCommand.test.ts:203-235` (`'does not show cost when pricing is not configured'`): asserts `result.content` does not contain `'Estimated cost'` when no `modelPricing` is set. Correct negative-path coverage.
- `statsCommand.test.ts:240+` (`'shows cost per model when multiple models have pricing'`): only the setup is visible in the head-200, but the structure (two models, two distinct pricings) is the right matrix.
- The `toModelMetrics` helper hints at a recent refactor where `ModelMetrics` grew a `bySource` map wrapping the raw `ModelMetricsCore`. If that's a new shape introduced elsewhere in the PR (not in the head-200 of the diff), the production cost-summing code needs to walk `bySource` and sum across keys, not just read top-level. Worth verifying.
- Sketchy bit: the `(contextWithPricing.services.settings as unknown as Record<string, unknown>)['merged'] = { modelPricing: ... }` pattern is a common test seam for mocking settings, but it relies on the production code reading from `.merged` rather than a typed accessor. If the production calculator reads from `settings.get('modelPricing')` instead, the test setup will not actually inject the pricing and the test would pass for the wrong reason. Worth a sanity check that the asserted `'$0.9000'` is actually produced from the injected map and not from a default.

## Verdict

**merge-after-nits**

## Reasoning

The user-facing surface (`modelPricing` settings entry + `/stats model` cost line) is the right shape and the unit tests cover the three load-bearing cases (configured, unconfigured, multi-model). Three quick follow-ups before merge: (1) add a test for the partial-config path where only one of `inputPerMillionTokens` / `outputPerMillionTokens` is set, since both fields are optional in the schema; (2) confirm the `toModelMetrics` helper's `bySource: { [MAIN_SOURCE]: core }` assumption matches how production sub-agent metrics are actually keyed (or add a multi-source test); (3) verify that `settings.merged.modelPricing` is the actual production read path used by the cost calculator, not just a test-shim that happens to work. The pricing math in the existing test is correct and the negative-path coverage is solid.
