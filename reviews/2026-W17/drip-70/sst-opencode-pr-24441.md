# sst/opencode#24441 — fix(zen): stop double-counting reasoning_tokens in oa-compat usage

- **Repo**: sst/opencode
- **PR**: [#24441](https://github.com/sst/opencode/pull/24441)
- **Head SHA**: `1c510292d5ea`
- **Author**: tiffanychum
- **Base**: `dev`
- **Size**: +91 / −3, 2 files

## Context

Per the OpenAI chat-completions spec, `usage.completion_tokens` is the *total*
output bucket and already includes `completion_tokens_details.reasoning_tokens`
as a sub-component. The console's zen route was treating them as additive and
billing both, which over-charges every reasoning-capable OAI-compat provider.

## Change

`packages/console/app/src/routes/zen/util/provider/openai-compatible.ts:61-86`
splits the previous `outputTokens` into a `completionTokens` raw value and a
clamped `reasoningTokens`, then returns:

```ts
outputTokens: completionTokens - (reasoningTokens ?? 0),
reasoningTokens,
```

with a `Math.min(reasoningTokensRaw, completionTokens)` clamp to defend against
providers (the comment names Moonshot Kimi K2.6) that report
`reasoning_tokens > completion_tokens` — so the invariant
`outputTokens + reasoningTokens === completion_tokens` always holds and we
charge no more than the upstream API billed.

`packages/console/app/test/zen-usage.test.ts` is a new file with two tests:
the "subtracts" base case (`prompt=22, completion=1226, reasoning=790` →
output 436, reasoning 790) and the "Hi" clamp case (`completion=77,
reasoning=78` → output 0, reasoning 77). Both exercise `oaCompatHelper`
directly with a Kimi-shaped provider context, which is good — the only place
the fix lives is also the only place exercised.

## Strengths

- Correct framing of the spec: the inline comment explicitly cites that
  downstream cost calc bills `outputCost + reasoningCost` separately, so the
  subtraction here is the right layer.
- The `Math.min` clamp is the right defensive choice (`floor at 0` for
  `outputTokens`, exact-match invariant for the sum) — the alternative of
  passing reasoning unclamped would over-charge by 1 token in the reporter's
  example, which compounds across many small turns.
- Test names tie back to the user-visible reproducer ("Hi" example).

## Risks / nits

1. The fix lives in the `oa-compat` helper but the sibling
   `openai.ts` helper (the native OpenAI provider) is untouched. Confirm in
   the PR thread that native OpenAI's `normalizeUsage` already does the
   subtraction (or doesn't need to) — otherwise this is half the bug. The
   diff doesn't show that file; one paragraph in the PR body asserting the
   parity check would close the loop.
2. The `cacheReadTokens` path at the same site still does
   `inputTokens - (cacheReadTokens ?? 0)` for `inputTokens`. That's a separate
   concern but worth a comment that the subtraction direction is intentional
   (cache reads are billed at a different rate, so they're peeled off input).
3. No regression test for the `reasoning_tokens === undefined` path
   (non-reasoning model). Easy add: `outputTokens` should equal
   `completion_tokens` exactly.

## Verdict

**merge-after-nits** — the fix is correct and the clamp is well-reasoned.
Worth one comment in the PR confirming the native-OpenAI helper has parity
behavior before merging, and a third test for the no-reasoning baseline.

## What I learned

`completion_tokens` already includes `reasoning_tokens` is one of those
spec details that's easy to miss because the OpenAI docs phrase it as
"completion_tokens_details" (sounds like a sibling, reads like a child).
Multiple OAI-compat layers in this repo's history have gotten it wrong;
the right fix lives at the *normalize* layer, not the cost-calc layer,
because every downstream consumer (billing, UI, telemetry) trusts that
`outputTokens + reasoningTokens` is the user-facing total. Clamp at the
boundary, not inside the math.
