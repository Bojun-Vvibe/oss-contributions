# Review: sst/opencode #25292 — fix: Update regex for maximum context length error to support sglang

- **PR**: https://github.com/sst/opencode/pull/25292
- **Author**: koush (Koushik Dutta)
- **Head SHA**: `30de3805ff199e723919ea34a5d7fd082c771dc3`
- **Base**: `dev`
- **Files**: `packages/opencode/src/provider/error.ts` (+1/-1)
- **Verdict**: **merge-as-is**

## Reasoning

This is a one-character regex widening at `provider/error.ts:15` inside the
`OVERFLOW_PATTERNS` table. The existing pattern `/maximum context length is
\d+ tokens/i` matched the OpenRouter / DeepSeek / vLLM phrasing
("…maximum context length is 32000 tokens…") but not the SGLang phrasing
("…maximum context length **of** 202752 tokens…"). The diff swaps `is`
for the alternation `(is|of)` and updates the trailing comment to add
`sglang` to the list of providers covered.

Mechanical correctness:

- The alternation is grouped with `()` rather than `(?:…)`, so it does
  introduce a capture group. None of the surrounding `error.ts` logic
  uses positional capture groups from this regex (it's an `OVERFLOW_PATTERNS`
  membership check via `.some(p => p.test(msg))`, not a `.match()` extraction),
  so the capture is harmless. Not worth blocking on.
- Both phrasings remain anchored on the literal `\d+ tokens` tail, so the
  pattern can't accidentally match an unrelated "maximum context length is
  bounded" sentence somewhere upstream.
- The author explicitly cites the verbatim SGLang error string in the PR
  body — a 189-token overage on a 202752-limit model — and tested locally
  with a modified checkout. Reproduction is trivial; trust signal is high.

The PR body cleanly references issue #25231 and is properly scoped to a
single file. No tests added, but `OVERFLOW_PATTERNS` is a flat lookup
table where the existing entries don't have per-pattern unit coverage
either, so adding one fixture for SGLang would be a refactor request,
not a blocker for this specific change.

## Suggested follow-ups

- Optional: convert the capture group to a non-capturing `(?:is|of)`
  for marginal hygiene if any future caller starts using `.exec()`
  on these patterns. Pure stylistic cleanup, not blocking.
- Optional follow-up PR: add a small fixture test asserting each entry
  in `OVERFLOW_PATTERNS` matches at least one canonical real-world
  error string from the provider it's tagged for. Would make future
  additions like this self-documenting.
