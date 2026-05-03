# Review: QwenLM/qwen-code #3814 — fix(core): prevent auto-memory recall from blocking main request

- PR: https://github.com/QwenLM/qwen-code/pull/3814
- Head SHA: `bfeb9ce9caf154d94c42f73bed3734798dd30a2e`
- Author: B-A-M-N (John London)
- Size: +1128 / -37

## Summary
Fixes #3759 — the auto-memory recall side-query had a 5s
`AbortSignal.timeout` that fired every turn, and because the main
request awaited the full recall promise (timeout + heuristic
fallback), every turn was delayed by ~5s. This PR shortens the
selector timeout to 2s and races the recall against a 2.5s deadline
on the main request path. **Note**: file list is identical to PR
#3815 — they appear to share a base branch and substantially
overlap.

## Specific citations
- `packages/core/src/memory/relevanceSelector.ts:90-92` — drops
  `selectRelevantAutoMemoryDocumentsByModel` timeout from 5000 to
  2000 ms. Selector is a ranking task; 2s is still generous for a
  fast-model call. Reasonable.
- `packages/core/src/core/client.ts:127-156` — defines
  `EMPTY_RELEVANT_AUTO_MEMORY_RESULT` and the
  `resolveAutoMemoryWithDeadline()` helper. The helper races the
  caller-supplied recall promise against a 2.5s `setTimeout` and
  resolves to the empty result if the deadline wins. Clean
  Promise.race pattern. The 2.5s value is intentionally `selector
  timeout (2s) + 0.5s slack`, which matches the design intent.
- `packages/core/src/core/client.ts:802-835` and `:1092-1138` —
  call sites in `sendMessageStream` switched from awaiting
  `recall()` directly to awaiting `resolveAutoMemoryWithDeadline()`.
- `packages/core/src/core/client.test.ts:1501-1601` — two tests
  using `vi.useFakeTimers()`:
  - "should not block the main request when auto-memory recall is
    slow" sets recall to a 10s `setTimeout`, advances 3s, asserts
    `mockTurnRunFn` was called WITHOUT the slow memory content
    (`expect.not.arrayContaining([...slow memory result...])`).
  - "should include auto-memory prompt when recall completes within
    deadline" uses `mockResolvedValue` for an immediate result and
    asserts the memory IS included.
  Both assertions are direct and target the new code path. Good.
- `packages/core/src/utils/retry.ts:+166/-9` and
  `retry.test.ts:+515/-19` — substantial retry changes not described
  in the PR body. Likely belong to a separate concern.
- `packages/core/src/core/geminiChat.ts:8 lines added` and
  `geminiChat.test.ts:+174` — also not mentioned in the PR
  description.

## Verdict
**needs-discussion**

Same concern as #3815: the diff is much wider than the description.
The auto-memory deadline fix itself looks correct and well-tested
(the fake-timer assertions are tight). But +515 lines of retry
tests and 178 lines of `geminiChat.test.ts` need to be attributed
or split out. Ask the author whether #3814 and #3815 should be
collapsed into one PR (they share file content) or rebased into
disjoint stacks. As-is, merging both creates conflicting overlapping
history.
