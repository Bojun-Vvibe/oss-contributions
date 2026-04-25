# anomalyco/opencode#24384 — fix(provider): respect configured output limit

- **Repo:** anomalyco/opencode
- **Author:** pascalandr (Pascal André)
- **Head SHA:** `5b8599ba` (5b8599bafa6fdcaf54c821828b35d51de6584318)
- **Size:** +39 / -1

## Summary
`maxOutputTokens` was capping configured output budgets at `OUTPUT_TOKEN_MAX`
via `Math.min`, silently shrinking large models that legitimately advertise
65k+ output. PR replaces with `model.limit.output > 0 ? model.limit.output :
OUTPUT_TOKEN_MAX`, falling back only when no limit is configured.

## Specific references
- `packages/opencode/src/provider/transform.ts:1037-1038` @ `5b8599ba` — single-line behavioral change. `Math.min(model.limit.output, OUTPUT_TOKEN_MAX) || OUTPUT_TOKEN_MAX` → `model.limit.output > 0 ? model.limit.output : OUTPUT_TOKEN_MAX`.
- `packages/opencode/test/provider/transform.test.ts:5-44` @ `5b8599ba` — adds two test cases: `output: 65_536` honored, `output: 0` falls back. Good positive + fallback coverage.

## Observations
1. **`OUTPUT_TOKEN_MAX` no longer acts as a ceiling**: the previous code intentionally bounded output at the constant. If `OUTPUT_TOKEN_MAX` exists to protect against pathological provider misconfig (e.g. a model spec that accidentally claims `output: 10_000_000`), this PR removes that guardrail entirely. Worth a one-line note in the PR description on whether `OUTPUT_TOKEN_MAX` is now purely a fallback default. If so, consider renaming to `OUTPUT_TOKEN_DEFAULT` to match its new role.
2. **Negative-value handling**: `> 0` correctly treats negative outputs as "unconfigured", but `model.limit.output` is typed as `number` — would `Number.isFinite(model.limit.output) && model.limit.output > 0` be even safer against an `Infinity` or `NaN` slip-through from a provider catalog?
3. **No test for very large values**: a test asserting `output: 1_000_000` returns `1_000_000` (not capped) would lock in the new contract and prevent a future "let's reintroduce the ceiling" regression.

## Verdict
**merge-after-nits** — semantic shift in `OUTPUT_TOKEN_MAX`'s role deserves either a comment or a rename; one extra test pinning the no-ceiling contract is cheap insurance.
