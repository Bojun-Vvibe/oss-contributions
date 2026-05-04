# sst/opencode #25180 — fix: enable auto-compaction for sub-agents and improve context overflow detection

- **Head SHA reviewed:** `0d670f143e40b74a51289366a7abd21e96a4a1a1`
- **Size:** +29 / -0 across 2 files
- **Verdict:** request-changes

## Summary

Two intertwined fixes for sub-agent context overflow:

1. `packages/opencode/src/provider/error.ts:28-30` — extends
   `OVERFLOW_PATTERNS` with three new regexes covering z.ai GLM,
   generic "max tokens exceeded", and "input too large for model"
   error strings.
2. `packages/opencode/src/session/processor.ts:546-568` — adds a
   pre-stream proactive compaction check: estimate the token count of
   the outgoing payload (`system + messages`), and if it's ≥ 85 % of
   `input.model.limit.context`, skip the API call entirely and return
   `"compact"` to drive the existing compaction path.

## What I checked

- `error.ts:28` `/token limit exceeded/i` — extremely broad. This
  string appears in plenty of *non-overflow* contexts: rate-limit
  responses ("daily token limit exceeded"), billing errors, org-cap
  errors. Treating those as overflow and silently triggering
  compaction will shrink the user's session in response to a billing
  problem. Recommend tightening to something like
  `/(?:context|input)\s+token limit exceeded/i` or anchoring around
  known z.ai phrasing.
- `error.ts:29` `/max.*tokens.*exceed/i` — the unbounded `.*` between
  `max` and `tokens` will match `max output tokens exceeded`, which is
  a *generation cap*, not a context overflow. Compaction won't help
  there and will throw away history needlessly. Suggest restricting
  to `max input tokens` / `max context` variants.
- `error.ts:30` `/input.*too.*large.*model/i` — same problem; `.*` is
  greedy and may match unrelated provider error prose. Anchor or
  narrow.
- `processor.ts:550-568` — the pre-stream estimate is the right shape,
  and the 85 % threshold matches the docs convention for the existing
  reactive path. Two concerns:
  - `JSON.stringify([...(streamInput.system ?? []).map((s) => s), ...streamInput.messages])`
    — the `.map((s) => s)` is a no-op; drop it. More importantly,
    `JSON.stringify` of the message array includes structural overhead
    (keys, quotes, brackets) that `Token.estimate` will count, which
    over-estimates token usage and will trigger compaction earlier than
    intended. Estimating against the concatenated *content* strings
    (text parts only) would be closer to what the provider counts.
  - `contextLimit > 0` is a sensible guard, but for sub-agents whose
    model entry returns `limit.context = 0` this proactive path is
    silently skipped — exactly the case the PR title calls out as
    broken. Worth a slog.warn or fallback to a global default to keep
    the safety net.
- The PR body notes sub-agents specifically; I don't see a code change
  that *enables* compaction for sub-agents (e.g. a flag flip). The fix
  appears to rely on the proactive check firing before the parent's
  compaction-disabled path is taken. A short comment in `processor.ts`
  describing why this works for sub-agents (and a regression test
  spawning a sub-agent with a small `limit.context`) would make the
  intent legible and prevent future drift.

## Risk

Medium. The new regexes are over-broad and will false-positive on
billing/rate-limit errors, leading to surprise compactions. The
pre-stream estimator is conservative in the right direction but
slightly hot due to JSON-stringifying the entire message array on
every turn (cost grows with conversation length).

## Recommendation

Request changes:

1. Tighten the three new `OVERFLOW_PATTERNS` regexes so they don't
   match output-token caps or rate-limit messaging.
2. Drop the no-op `.map((s) => s)` and switch the estimator input to
   plain text content rather than stringified JSON.
3. Add a regression test (or at least a `slog` field) for the
   `contextLimit === 0` sub-agent path so the bug-fix premise is
   verifiable.
4. Consider caching `preStreamTokens` per turn to avoid re-estimating
   on retries within the same Effect generator.
