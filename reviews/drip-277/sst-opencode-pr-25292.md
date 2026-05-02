# Review — sst/opencode#25292

- PR: https://github.com/sst/opencode/pull/25292
- Title: fix: Update regex for maximum context length error to support sglang
- Head SHA: `30de3805ff199e723919ea34a5d7fd082c771dc3`
- Size: +1 / −1 across 1 file
- Verdict: **merge-as-is**

## Summary

Single-line widening of one of the OVERFLOW_PATTERNS regexes in
`packages/opencode/src/provider/error.ts` so that sglang's variant
("maximum context length **of** N tokens") classifies the same way as
the existing OpenRouter / DeepSeek / vLLM variant ("maximum context
length **is** N tokens"). Without this, sglang-backed providers raise
a generic provider error instead of being routed through opencode's
overflow-handling path (truncation / context warning).

## Evidence

- `packages/opencode/src/provider/error.ts:15` — the pattern changes from
  `/maximum context length is \d+ tokens/i` to
  `/maximum context length (is|of) \d+ tokens/i`. The trailing comment
  is updated to add `sglang` to the list of providers it covers.
- The capture group `(is|of)` is non-referenced; nothing downstream
  reads `match[1]`, so introducing the group has no behavioural side
  effect.
- The other ten OVERFLOW_PATTERNS entries are unchanged, which is
  correct — sglang's wording was the only outlier.

## Notes

The capture could be a non-capturing group `(?:is|of)` for
micro-cleanliness, but it makes zero functional difference and it's a
1-line drive-by fix. The PR is the smallest possible diff that does
the right thing; ship it.
