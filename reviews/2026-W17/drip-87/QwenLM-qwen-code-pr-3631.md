# Review — QwenLM/qwen-code#3631: feat(stats): add model cost estimation

- **Repo:** QwenLM/qwen-code
- **PR:** [#3631](https://github.com/QwenLM/qwen-code/pull/3631)
- **Author:** B-A-M-N (John London)
- **Head SHA:** `981d30a240c3acabc43591e1b1ffe3a3d9ee6d5a`
- **Size:** +733 / −25 across 10 files
- **Verdict:** `merge-after-nits`

## Summary

Adds an optional `modelPricing` settings object (`Record<string, { inputPerMillionTokens?: number; outputPerMillionTokens?: number }>`) at `packages/cli/src/config/settingsSchema.ts:958-974`, a new `costCalculator.ts` (33 lines impl, 205 lines test) that multiplies prompt/candidate token counts against the configured rate, and threads the result into `/stats model` via `statsCommand.ts` (+11) and `ModelStatsDisplay.tsx` (+33). When pricing is absent for the active model, the existing display path is untouched. Related to issue #3585.

## Technical assessment

The schema entry at `settingsSchema.ts:958-974` correctly declares `requiresRestart: false`, `showInDialog: false`, and a typed `default: undefined` cast through the `Record<string, ...>` shape. That keeps it out of the settings UI (this is a power-user knob) while still validating the JSON. The `description` example uses real qwen3-coder pricing (`0.30` input / `1.20` output per million) which is a nice touch.

The `costCalculator.ts` module is the right size — pure function, no I/O, 33 lines, 205 lines of test (`costCalculator.test.ts`). The test surface visible in the diff covers (a) cost shown when pricing exists, (b) cost not shown when pricing absent, (c) the lookup honors model-name keys. The `prompt=1000000` × `0.30/M` + `candidates=500000` × `1.20/M` = `$0.9000` arithmetic at `statsCommand.test.ts:222` is correct (`0.30 + 0.60 = $0.90`), so the unit math is verified.

The settings-injection pattern in tests at `statsCommand.test.ts:165-176` uses `(contextWithPricing.services.settings as unknown as Record<string, unknown>)['merged']` — that's a code-smell cast but it's confined to test code and the production path reads through a typed accessor.

`modelConfigUtils.ts:+10/−2` adjusts the model lookup to pass through the pricing-key resolution (probably normalizing `qwen3-coder` vs `Qwen/qwen3-coder` provider-prefixed names). `modelConfigUtils.test.ts:+65/−20` covers it.

## Nits worth addressing pre-merge

1. **Cached tokens not credited.** `ModelMetrics` carries a `cached` token count separately from `prompt`. Most providers (Anthropic, OpenAI) bill cached input at a fraction (often 10–25%) of full input rate. The current calculator just uses `prompt` and `candidates` and does not deduct or rebate `cached`. Either (a) add `cachedReadPerMillionTokens` and `cachedWritePerMillionTokens` to the schema and apply them, or (b) explicitly document in the description that the estimate ignores cache discounts and may overstate cost for cache-heavy sessions. The latter is a one-line fix; the former is the right long-term shape.

2. **No reasoning-token line item.** `ModelMetrics.tokens.thoughts` is tracked separately. For reasoning-capable models (qwen3-thinking, deepseek-r1) the thoughts tokens are billed at output rate by most providers but at a separate rate by some. Currently `thoughts` is silently uncounted. Either include it in the `candidates` multiplier or add a `reasoningPerMillionTokens` knob.

3. **Per-currency assumption.** Hard-coded `$` prefix in `Estimated cost: $0.9000` (verified at `statsCommand.test.ts:225`). PR body's "Not covered: currency conversion" caveat is honest, but the symbol should at least be parametric — `currencySymbol?: string` defaulting to `$` — so a CNY-paying user can render `¥`. Tiny change.

4. **Format precision.** `$0.9000` shows 4 decimal places. For long sessions this is fine; for one-shot completions `$0.0001` and below will all show as `$0.0000`. Consider `Math.max(4, -Math.floor(Math.log10(cost)))` decimals or a `toLocaleString` with `maximumSignificantDigits: 3`.

5. **Schema discoverability.** `showInDialog: false` means users find this only by reading docs or settings JSON. A `qwen pricing example` slash command or a `--print-pricing-template` CLI flag would help adoption. Out of scope for this PR but worth tracking.

## Verdict rationale

`merge-after-nits` — the implementation is small, well-tested, opt-in, and correctly leaves the no-pricing path untouched. The cached-token and reasoning-token gaps are real (today's qwen3-thinking sessions can have 30%+ thoughts tokens) but addressable in a follow-up if not blocking this merge. Cost-estimation features are easy to ship in v0 form and hard to make accurate; this is a reasonable v0.
