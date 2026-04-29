# QwenLM/qwen-code#3737 — `fix(core): preserve reasoning_content in rewind, compression, and merge paths (#3579)`

- URL: https://github.com/QwenLM/qwen-code/pull/3737
- Head SHA: `480f9e71b4c4`
- Author: fyc09
- Size: +48 / -363 across 8 files
- Closes the #3579 series (after #3590, #3682)

## Summary

Three remaining paths still silently dropped DeepSeek-class
`reasoning_content` from history:

1. **Rewind / double-ESC** — `AppContainer.tsx` called
   `geminiClient.stripThoughtsFromHistory()` after truncating
   to the rewind point.
2. **Chat compression** — `chatCompressionService.ts` built the
   acknowledgment model turn with no thought part, so converters
   that key off the presence of *any* thought in a model turn
   broke the reasoning chain across the compression boundary.
3. **Idle/merge** — the `stripThoughtsFromHistory` and
   `stripThoughtsFromHistoryKeepRecent` helpers themselves were
   load-bearing for a use case the project no longer wants to
   support; this PR retires them outright.

## Specific references

`packages/cli/src/ui/AppContainer.tsx:1739-1745` (post-patch):

```tsx
- // 3. Truncate API history and strip stale thinking blocks
+ // 3. Truncate API history to the target point.
+ // Do NOT strip thought parts — reasoning models (e.g. DeepSeek)
+ // require reasoning_content continuity across all turns ...
  geminiClient.truncateHistory(apiTruncateIndex);
- geminiClient.stripThoughtsFromHistory();
```

`packages/core/src/core/client.ts:204-210` removes the
`stripThoughtsFromHistory()` pass-through on `GeminiClient`.

`packages/core/src/core/geminiChat.ts:792-919` (the deletion block)
removes both `stripThoughtsFromHistory()` and
`stripThoughtsFromHistoryKeepRecent(keepTurns)` from `GeminiChat`,
and `geminiChat.test.ts:2039-2227` removes their dedicated tests.
This is a real public-API surface reduction.

`packages/core/src/services/chatCompressionService.ts:265-292` —
the new logic detects whether any preserved model turn carries
`thought: true` and, if so, appends a sentinel thought part to
the compression acknowledgment model turn:

```ts
const hasThoughtContentInKeep = historyToKeep.some(
  (content) =>
    content.role === 'model' &&
    content.parts?.some((part) => 'thought' in part && part.thought),
);

const ackParts: Part[] = [
  { text: 'Got it. Thanks for the additional context!' },
];
if (hasThoughtContentInKeep) {
  ackParts.push({ text: ' ', thought: true });
}
```

## Design analysis

The PR implements a clean policy reversal: instead of stripping
thoughts on three control paths (rewind, compression-ack, idle),
preserve them and remove the strippers entirely. That matches
the framing in #3579 — DeepSeek-class providers require an
unbroken `reasoning_content` chain across the *entire* assistant
sequence, so any helper that selectively drops thoughts is a
loaded gun.

Two design points worth noting:

1. **Sentinel thought part with `text: ' '`.** The acknowledgment
   only carries a thought part *when* prior turns had one. The
   chosen sentinel is a single-space text with `thought: true`.
   That works for downstream converters that key off "is there a
   thought part on this turn" — but it's brittle if any consumer
   ever asserts non-empty thought content. A short comment
   right above the `ackParts.push` line explaining "sentinel,
   intentionally minimal" would help future maintainers.
2. **API surface removal vs deprecation.** Deleting public
   methods on `GeminiClient` and `GeminiChat` is fine for an
   internal/CLI-only surface, but if anything in
   `packages/cli`-adjacent extension code calls them, this is
   a hard break. A `rg "stripThoughtsFromHistory"` across the
   whole repo (already implied by the test cleanup) plus a note
   in the PR body would close the loop.

The test cleanup is mechanical: every `stripThoughtsFromHistory: vi.fn()`
mock entry is removed across `client.test.ts`, `config.test.ts`,
and `geminiChat.test.ts`. No assertion logic for the new behavior
was added in `geminiChat.test.ts` itself; the new behavior is
exercised through `chatCompressionService` (presumed to have its
own tests, not visible in the diff).

## Risks

- Compression test coverage: the diff doesn't show a new test in
  `chatCompressionService.test.ts` pinning the
  `hasThoughtContentInKeep ? thought-part-present : not` branch.
  This is the only new code path; it should have a direct test.
- The `text: ' '` sentinel is invisible in user-facing logs but
  may surface in trace dumps. Worth confirming no downstream
  formatter trims whitespace-only text and drops the thought part
  with it.
- Prior fixes #3590 and #3682 closed the resume/idle and
  model-switch paths. After this PR, any *new* control path that
  wants to mutate history needs to learn this lesson without the
  guardrail of helpers — the helpers are gone. A short
  `CHANGELOG`/comment block in `geminiChat.ts` explaining
  *why* these helpers no longer exist would prevent re-introduction.

## Verdict

`merge-after-nits`

## Suggested nits

- Add at least one test in `chatCompressionService.test.ts`
  pinning both branches of the `hasThoughtContentInKeep` check.
- Add a short tombstone comment in `geminiChat.ts` (where the
  methods used to live) saying "intentionally removed in #3737
  — reasoning_content must remain contiguous; do not reintroduce".
- Confirm in PR body that `rg stripThoughtsFromHistory` returns
  zero hits across the whole repo after this PR.
