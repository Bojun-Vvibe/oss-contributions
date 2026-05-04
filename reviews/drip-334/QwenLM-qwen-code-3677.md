# QwenLM/qwen-code#3677 — fix(openai): parse MiniMax thinking tags

- **Head SHA:** `814ea0d60bb1a8021fb75ac4ecaed4199de4825e`
- **Author:** community
- **Size:** +726 / −7, 13 files

## Summary

Two distinct concerns wrapped into one PR:

1. **MiniMax thinking-tag parser** — adds an OpenAI-compatible provider gated to `*.minimaxi.com` endpoints that converts inline `<think>...</think>` / `<thinking>...</thinking>` content into Gemini `thought: true` parts. New files: `provider/minimax.ts`, `provider/minimax.test.ts`, `provider/types.ts`, `provider/index.ts`, `responseParsingOptions.ts`, `taggedThinkingParser.ts`, plus changes in `converter.ts`, `pipeline.ts`, `index.ts`, `types.ts`.

2. **UI fix** — `useGeminiStream.ts` strips leading blank chunks so empty `\n\n` deltas don't render as empty assistant/thought items.

## Specific citations

- `useGeminiStream.ts:110-112` — new helper `stripLeadingBlankLines(text)` using regex `/^(?:[ \t]*\r?\n)+/`. Correct: matches whitespace+newline runs, leaves intra-line whitespace alone.
- `useGeminiStream.ts:773-779` — guard in the assistant branch: when no pending gemini item exists AND `newGeminiMessageBuffer.trim().length === 0`, return early without flushing or starting a new pending item. Prevents creating empty `gemini` history items from leading blank chunks. Reassigns `newGeminiMessageBuffer = stripLeadingBlankLines(newGeminiMessageBuffer)` (`:780`) so the first non-blank chunk doesn't carry the leading newlines.
- `useGeminiStream.ts:855-872` — same pattern for thoughts: trim-empty early return + `stripLeadingBlankLines` + reconstitute `thoughtToMerge = { ...eventValue, description: newThoughtBuffer }`. Then `mergeThought(thoughtToMerge)` instead of `mergeThought(eventValue)` on `:907` — important so the thought-state loading indicator gets the cleaned text.
- `useGeminiStream.test.tsx:1208-1278` — new test simulating `\n\n` then `'哈哈'` chunks; asserts `pendingHistoryItems` stays `[]` after the blank chunk, then becomes `[{ type: 'gemini', text: '哈哈' }]`. Uses `vi.useFakeTimers()` and `vi.advanceTimersByTime(60)` to drive the throttle. Also mocks `findLastSafeSplitPoint` to return `s.length` (`:152-154`) so split logic doesn't interfere.
- `useGeminiStream.test.tsx:1336-1407` — parallel test for thoughts: `\n\n` then `'Thinking'`; asserts `result.current.thought` stays null, then becomes `{ description: 'Thinking' }`. Good coverage.
- `converter.test.ts:1912+` — new describe block "OpenAI -> Gemini tagged thinking content" with both streaming and non-streaming cases for `<think>internal reasoning</think>final answer` -> two parts (thought + text).

## Verdict: `merge-after-nits`

Both fixes are correct and well-tested. Concerns:

1. **PR scope.** The blank-chunk fix in `useGeminiStream.ts` is logically independent from the MiniMax thinking-tag parser. The blank-chunk fix is a UI quality bug that affects all providers; the MiniMax parser is a provider-specific feature. Recommend splitting into two PRs so the UI fix can land independently and be backported if needed.
2. **Provider detection by URL substring.** `*.minimaxi.com` matching is presumably done in `provider/minimax.ts` (not visible in this diff slice). URL-based provider gating is fragile when users front the API behind a proxy/gateway; consider an explicit config flag (`provider: "minimax"`) as the primary gate and URL sniffing as a fallback.
3. **`stripLeadingBlankLines` regex** matches `[ \t]*\r?\n` repeatedly but doesn't handle `\u00A0` (non-breaking space) or other Unicode whitespace. Probably fine since the upstream is server-emitted ASCII whitespace, but consider `/^\s*?\n+/` if you want to be defensive. (Note: `/^\s+/` would over-match if the first line is meant to start with leading spaces.)

The thought-merge fix at `:907` is subtle and correct; nice catch.
