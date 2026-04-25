# sst/opencode#24336 — fix(session): clamp token usage counts
**SHA**: `e7012ca2` · **Author**: pascalandr · **Size**: +40/-7 across 2 files

## What
Wraps every call site that reads numeric token counts from the
LanguageModel usage object with a new `nonNegative` helper
(`Math.max(0, safe(value))`). The motivating case: the AI SDK v6
normalisation now folds cached input tokens into `inputTokens` for all
providers, and the existing code subtracts `cacheRead + cacheWrite`
from `inputTokens` to recover the non-cached count. When a provider
double-counts or returns slightly-off cache numbers (Anthropic's
`cacheCreationInputTokens` lives in metadata and may diverge from the
SDK's `cacheWriteTokens`), `inputTokens - cacheRead - cacheWrite` can
go negative — which then poisons cost calculations and any UI showing
input tokens.

## Specific cite
- `packages/opencode/src/session/session.ts:304` is the new
  `nonNegative = (value: number) => Math.max(0, safe(value))` helper.
  Worth noting it composes with the existing `safe()` (which
  NaN-coerces to 0); the order matters because `Math.max(0, NaN)`
  returns `NaN`.
- The critical line is `:330` —
  `const adjustedInputTokens = nonNegative(inputTokens - cacheRead - cacheWrite)`.
  This is the actual bug fix; the others
  (`:304-:307,:338`) are defense-in-depth applied uniformly so future
  refactors don't accidentally let a negative leak through.
- The new test at `test/session/compaction.test.ts:1918-1947` is
  exactly the right shape: input=100, cacheRead=200, cacheCreation=150
  → asserts `tokens.input === 0` and `tokens.output === 0` (output
  also clamps because reasoning > output in the synthetic case).

## Verdict: merge-as-is
Surgical, well-tested, and the test case directly exercises the AI
SDK v6 + Anthropic metadata interaction described in the inline
comment. The defensive uniformity (clamping all six fields, not just
the one that caused the bug) is the right call here; it costs nothing
and removes a whole class of off-by-cache regressions.
