# sst/opencode#24572 — fix(opencode): prevent negative cost when cache tokens exceed input

- PR: https://github.com/sst/opencode/pull/24572
- Head SHA: `97050b79`
- Base: `dev`
- Diff: +29 / -1 across `packages/opencode/src/session/session.ts` (+1/-1) and `packages/opencode/test/session/compaction.test.ts` (+28)

## What it does
Closes #22618. Fixes the long-standing "$ spent" sidebar regression where switching from a paid model (e.g. Claude 3.5 Opus) to a free model (e.g. MiniMax 2.5) caused the running cost total to *decrease*. Root cause is in `getUsage` at `packages/opencode/src/session/session.ts:298-301`: the `safe(value)` helper only filtered for `Number.isFinite(value)`, but `adjustedInputTokens = inputTokens - cacheReadTokens - cacheWriteTokens` can go negative whenever the provider reports `inputTokens` as the *new* tokens (excluding cache) rather than the *total* (including cache). Negative tokens × non-zero unit cost = negative session cost contribution.

The fix is a one-character logical addition: `if (!Number.isFinite(value) || value < 0) return 0`.

## Strengths
- Correct, minimal, non-invasive fix at the right layer. The reporting inconsistency is a provider-side property `getUsage` cannot prevent — clamping to 0 here is the right policy and matches what every billing system does in practice ("never refund").
- The new regression test at `compaction.test.ts:2092-2119` constructs the exact pathological shape (`inputTokens: 100`, `cacheReadTokens: 200`, `cost.input: 10`) and asserts both `result.tokens.input === 0` and `result.cost === 0`. That pins both the token-clamp and the cost-clamp invariants — without it, a future refactor that moved the clamp from `safe()` into a different helper could silently regress.
- PR description correctly identifies the asymmetry between providers (some report total, some report new) as the upstream issue and frames the fix as defense at the consumer.

## Concerns
- The clamp is now silent. If a provider starts reporting wildly negative deltas (e.g. due to a bug on their side), opencode will swallow that without any telemetry breadcrumb. A `debug`-level log when `value < 0` is observed would help diagnose provider regressions later without spamming users. Cheap to add.
- The `safe()` helper is only used inside `getUsage` — it's local. But naming `value < 0` as part of "safe" overloads the name. A reviewer reading just the function later may wonder if `safe` also clips above some max. Either rename to `safeNonNegative` or add a one-line comment: `// providers can report inputTokens as net-of-cache; clamp to 0 to prevent negative cost`.
- The test doesn't also cover `outputTokens < 0` or `cacheReadTokens < 0`. Those are equally possible under the same provider-reporting asymmetry, and `safe()` now clamps all of them. A `test.each` over the four token fields would lock down the contract uniformly.
- No test for the `outputTokens=undefined` + `cacheRead=200` shape, which exercises `safe(undefined ?? 0) - safe(200) = -200` through a different path. Probably handled by the same clamp, but worth pinning.

## Verdict
**merge-as-is** — one-line behavior fix at the correct layer, with a focused regression test that pins the bug shape from the linked issue. Suggestions above are nits and shouldn't block merge.

## What I learned
Token-accounting bugs in agent UIs almost always trace back to provider-reporting asymmetries (gross vs net of cache, prompt vs completion split, cache-write vs cache-read accounting). The defensive posture for the *consumer* is to clamp at the boundary (here, `safe()`) rather than try to detect which provider variant you're talking to — but the clamp should be loud (debug log) so genuine provider regressions still surface.
