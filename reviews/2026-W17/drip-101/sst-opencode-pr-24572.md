# sst/opencode PR #24572 — fix(opencode): prevent negative cost when cache tokens exceed input

- URL: https://github.com/sst/opencode/pull/24572
- Head SHA: `97050b795bfee208e2927c69f1fa7765e6f91f50`
- Diff: +29 / -1 across 2 files
- Author: external contributor (Felix-Hz)

## Context / Problem

`SessionNs.getUsage` computes `adjustedInputTokens = inputTokens -
cacheReadTokens - cacheWriteTokens` to avoid double-billing cache reads.
Providers disagree about whether `inputTokens` is *total* (cache included) or
*new-only*; for the new-only providers, the subtraction goes negative. When a
non-zero input price is configured (model definition or fallback), the
negative token count multiplied by price produces a *negative cost delta* that
gets added to the running session total, so the sidebar "$ spent" indicator
visibly *decreases* mid-session — most obvious when switching from a paid to a
free model. Closes upstream issue #22618.

## Design

One line, one branch:

```ts
// packages/opencode/src/session/session.ts:300-303
const safe = (value: number) => {
  if (!Number.isFinite(value) || value < 0) return 0
  return value
}
```

Because every token-count consumer (input, output, cache read, cache write,
adjusted input) flows through `safe()`, clamping at the helper is the
narrowest fix that covers every divergent-provider permutation without
having to patch each subtraction site. The fact that this helper already
existed and already swallowed `NaN`/`Infinity` is exactly why adding the
`< 0` arm here is the right edit.

Test at `test/session/compaction.test.ts:2092-2117` synthesizes the bug
shape (`inputTokens: 100`, `cacheReadTokens: 200`) and pins both
`result.tokens.input === 0` and `result.cost === 0`. Author claims the test
fails without the one-line fix and passes with it; the math
`safe(100 - 200) = safe(-100) = 0` confirms.

## Risks / nits

- Clamping silently *hides* the underlying provider-reporting disagreement
  rather than logging it. If a future provider regresses on token semantics,
  the sidebar will look correct but accounting will silently undercount. A
  `tracing.warn` (or equivalent) one-shot per session when `cacheReadTokens >
  inputTokens` would preserve visibility without spamming. Worth a follow-up
  issue, not a block.
- The PR doesn't add a per-provider note to docs — the maintainer probably
  knows which providers are new-only-reporters but the next reader of
  `getUsage` won't, and that asymmetry is the actual root cause. A two-line
  comment on the `safe` helper ("providers disagree on whether inputTokens
  includes cache reads — clamp negatives so accounting stays monotonic")
  would prevent the same bug from being reintroduced when someone "fixes"
  the apparent under-counting.
- Test asserts `result.cost === 0` but the model has `cost: { output: 0 }`,
  so the output side contributes nothing. Adding a small non-zero output
  cost would make the test additionally pin "we still bill the output side
  even when input goes negative-then-clamped".

## Verdict

`merge-after-nits` — the one-line clamp is correct and minimal, the test
shape is right, but the comment-on-helper and the optional-warn would lift
this to "no-future-regret".

## What I learned

A single normalization helper used at every numeric-input edge is a good
shape: when the next provider quirk shows up, you patch the helper rather
than every call site. The cost is that a silent clamp can hide a real
upstream contract drift — pair the clamp with a debug log if accounting
matters.
