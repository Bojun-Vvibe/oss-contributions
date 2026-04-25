# anomalyco/opencode #24367 — fix(zen): stop double-counting reasoning_tokens in oa-compat usage

- URL: https://github.com/anomalyco/opencode/pull/24367
- Head SHA: `bf15a1bbd11e3ec38752af3d80b8548e1965ba7d`
- Verdict: **merge-as-is**

## What it does

The OpenAI chat-completions usage spec is explicit:
`completion_tokens` already includes
`completion_tokens_details.reasoning_tokens`. Zen's
`oaCompatHelper.normalizeUsage` was reporting `outputTokens =
completion_tokens` and `reasoningTokens = reasoning_tokens` as separate
buckets, so downstream `calculateCost` (which bills `outputCost +
reasoningCost` at the same per-token rate) double-billed every reasoning
token. Reporter showed a `output: 1226, reasoning: 790, total: 2016`
console row for an upstream `completion_tokens=1226, reasoning_tokens=790`
call — exactly `1226 + 790`.

## Diff notes

`packages/console/app/src/routes/zen/util/provider/openai-compatible.ts:64-79`
renames the local to `completionTokens` and returns
`outputTokens: Math.max(0, completionTokens - (reasoningTokens ?? 0))`.
The clamp matters: Moonshot Kimi K2.6 reports `reasoning=78,
completion=77` for trivial inputs, which would otherwise produce a
negative `outputTokens` row.

The new test file `packages/console/app/test/zen-usage.test.ts` locks
the wire-level invariant with four cases — reporter's 1226/790 session,
the clamp case 77/78, no-reasoning passthrough, and parity with
`openaiHelper.normalizeUsage` for the same logical usage. The parity
test is the right anchor: it pins the OA-compat branch to whatever the
Responses helper does.

## Risk surface

- Strictly reduces `outputTokens` for any provider that returns
  `reasoning_tokens > 0` via OA-compat. That is the correction; bills
  drop to match what the upstream API actually charges.
- PR explicitly leaves `openaiHelper.normalizeUsage` untouched (it
  already subtracts but doesn't clamp). The Responses API doesn't
  exhibit the `reasoning > completion` quirk, so that scope deferral is
  defensible.

## Why this verdict

One-line fix for a real billing bug, with a parity-anchored test that
prevents the two helpers from drifting again. Ship.
