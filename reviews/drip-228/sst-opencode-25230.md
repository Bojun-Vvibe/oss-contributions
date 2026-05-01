# sst/opencode #25230 — Update regex for maximum context length error to support sglang

- **PR**: https://github.com/sst/opencode/pull/25230
- **Head SHA**: `30de3805ff199e723919ea34a5d7fd082c771dc3`
- **Files reviewed**: `packages/opencode/src/provider/error.ts`
- **Date**: 2026-05-01 (drip-228)

## Context

Error-classification regression. sglang's overflow message reads
`maximum context length **of** N tokens` rather than the
OpenRouter/DeepSeek/vLLM phrasing `maximum context length **is** N tokens`.
The pre-existing `OVERFLOW_PATTERNS` entry only covered the `is` variant,
so requests hitting the sglang ceiling were classified as generic
`Bad Request` instead of `ContextLengthExceeded`, which means the
auto-truncate / context-summarize recovery path never fired and the user
saw a raw upstream error.

## Diff (1 file, +1 -1)

`provider/error.ts:15`:

```diff
-  /maximum context length is \d+ tokens/i, // OpenRouter, DeepSeek, vLLM
+  /maximum context length (is|of) \d+ tokens/i, // OpenRouter, DeepSeek, vLLM, sglang
```

## Observations

1. **Correct minimal scope.** The fix is a single regex alternation
   `(is|of)` at exactly the spot where the existing pattern lives. The
   comment is updated to name sglang as a covered backend, which keeps
   the table self-documenting for the next person who has to extend it
   for a new inference server. No call-site code needs to change — the
   pattern is consumed by whatever generic `OVERFLOW_PATTERNS.some(...)`
   matcher already exists in this file.

2. **Anchor still strong enough.** The full phrase
   `maximum context length (is|of) \d+ tokens` is specific enough that
   it won't match unrelated messages — both `is` and `of` are gated by
   the surrounding fixed strings on both sides, so a stray
   `... maximum context length of the prompt ...` text wouldn't match
   (no `\d+ tokens` follow-up). Low risk of false positives.

3. **Nit (not blocking).** No regression test added. A two-line entry
   in the existing pattern test (matching the PR-body example
   `Bad Request: Requested token count exceeds the model's maximum
   context length of 202752 tokens. ...`) would pin the contract so a
   future "tighten the regex" cleanup can't silently drop sglang again.

## Verdict

**merge-after-nits** — ship the regex as-is, request a one-line test
covering the `of` arm so the behavior is locked in.

## What I learned

A regex table indexed by upstream-vendor phrasing is the right shape
for this kind of cross-backend error normalization, but it needs a
test arm per vendor or it silently regresses every time someone
"simplifies" the pattern. The cost of adding the arm is one line; the
cost of not having it is the user-visible `Bad Request` we just fixed.
