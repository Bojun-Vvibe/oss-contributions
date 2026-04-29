---
pr: sst/opencode#25012
sha: 2a83f7361ada4e181d1018212ede23fd71b55e9c
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix: make deepseek string check a bit looser

URL: https://github.com/sst/opencode/pull/25012
Files: `packages/opencode/src/provider/transform.ts`
Diff: 2+/2-

## Context

Two `model.api.id.includes("deepseek")` / `includes("deepseek-v4")`
gates at `transform.ts:200` and `:576` were case-sensitive substring
checks. Some upstream provider listings (Bedrock, OpenRouter
catalogue snapshots, Together's older entries) ship the same DeepSeek
model with mixed case (`DeepSeek-V4`, `Deepseek-r1`, etc.), and those
slipped past both:

- the assistant-reasoning-required normalization in
  `normalizeMessages`, which silently shipped a Deepseek request
  without the reasoning blocks the API enforces (→ 400)
- the `efforts.push("max")` extension at `:576` for V4, which silently
  dropped the `max` reasoning effort tier from the picker for any
  provider listing the model with a non-lowercase id

## What's good

- One-line lowercasing on both sides at `transform.ts:200,576`. The
  source-of-truth string `"deepseek"` / `"deepseek-v4"` was already
  lowercase, so `id.toLowerCase().includes(...)` is the textbook fix
  with no false negatives versus the prior check.
- Both call sites updated symmetrically — common bug pattern is to fix
  the message-normalization gate but forget the variants gate, this
  PR catches both.

## Nits / follow-ups

- None blocking. Long term, both call sites would benefit from being
  consolidated into a `isDeepseekId(id)` helper colocated with the
  other family predicates (the `isAnthropicFamily` helper introduced
  by #25003 in drip-184 is the precedent), but at 2 lines this is not
  worth gating on.

## Verdict

`merge-as-is` — minimal correct case-insensitive fix at both gates,
zero risk of regression for already-lowercase ids, addresses real
catalogue-source variance, no test infrastructure to update because
the bug class is "id casing" not "model behaviour".
