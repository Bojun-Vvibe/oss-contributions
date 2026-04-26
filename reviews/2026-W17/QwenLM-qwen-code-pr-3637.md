---
pr: 3637
repo: QwenLM/qwen-code
sha: a3c6c9a8b7895734e02140033a3378126672ea81
verdict: merge-as-is
date: 2026-04-26
---

# QwenLM/qwen-code #3637 — fix(core): preserve reasoning_content when merging consecutive assistant messages (#3619)

- **Author**: wenshao
- **Head SHA**: a3c6c9a8b7895734e02140033a3378126672ea81
- **Link**: https://github.com/QwenLM/qwen-code/pull/3637
- **Size**: +249 / -2 across 2 files: `converter.ts` (+31/-2) and `converter.test.ts` (+218/-0).

## Scope

When `mergeConsecutiveAssistantMessages` folds two adjacent `assistant` turns into one (typically because `cleanOrphanedToolCalls` removed the tool message between them), the second message's `reasoning_content` was silently dropped. For DeepSeek thinking-mode and other providers that require `reasoning_content` to remain populated across turns, this caused HTTP 400 on the next request. Fix: concatenate both turns' `reasoning_content`; when the merged result has reasoning but no visible text, emit `content: ""` instead of `content: null` (matches the pre-existing `processContent` rule for single messages).

## Specific findings

- `converter.ts:1321-1340` — extraction of `lastReasoning` and `currentReasoning` uses `'reasoning_content' in lastMessage` plus a cast to `ExtendedChatCompletionAssistantMessageParam`. The runtime guard plus `?? ''` handles both "field absent" and "field present but undefined" — defensive enough.
- `converter.ts:1338-1340` — `combinedReasoning = [lastReasoning, currentReasoning].filter(Boolean).join('')`. The `.filter(Boolean)` drops empty strings before joining, so when only one side has reasoning the join is a no-op. Correct. The empty-vs-empty case yields `combinedReasoning = ''`, which in turn keeps `content` going through the `combinedContent || null` path — preserving prior behaviour for non-reasoning merges.
- `converter.ts:1349` — `(lastMessage as ...).content = combinedContent || (combinedReasoning ? '' : null)`. The new branch only changes behaviour when `combinedContent` is falsy AND `combinedReasoning` is truthy — exactly the "reasoning-only merge" case that breaks Ollama and DeepSeek. Test `should use empty string instead of null for content when merged result has reasoning but no visible text (issue #3499)` at `converter.test.ts:163-188` covers this precisely.
- `converter.ts:1361-1364` — only writes `reasoning_content` back when `combinedReasoning` is truthy, so we don't introduce an empty `reasoning_content: ''` on plain merges. Good.
- `converter.test.ts:2024-2046` — happy path: both turns have `{thought: true}` parts, expects merged `content === 'visible 1visible 2'` and `reasoning_content === 'thinking step 1thinking step 2'`. Good.
- `converter.test.ts:2050-2079` — the highest-value regression: only the SECOND turn has reasoning, asserts the merged result keeps it. This is the exact failure mode that triggered #3619 and the test name calls it out. Good.
- `converter.test.ts:2084-2109` — only the FIRST turn has reasoning, asserts retention. Symmetric coverage with the previous test.
- `converter.test.ts:2116-2156` — most realistic test: includes an orphaned `functionCall` in turn 1 (which `cleanOrphanedToolCalls` would strip), forcing the merge path. Asserts `merged.tool_calls === undefined` (orphan stripped), `merged.reasoning_content === 'planreplan'`, `merged.content === 'visible 1visible 2'`. This validates the actual production code path that triggers the bug, not just the in-isolation merge. Excellent.
- `converter.test.ts:2161-2188` — Ollama compatibility: two reasoning-only turns merge to `content: ''`, NOT `null`. Linked to #3499 in the comment.
- `converter.test.ts:2191-2226` — three-message case: tests that the merge logic accumulates correctly across more than two turns. Asserts `content === 't1t2t3'` and `reasoning_content === 'r1r3'` (skips the missing `r2` from the middle turn). This is good coverage of the fold semantics.
- The PR has no behaviour change for the 2000+ lines of pre-existing tests — the diff at `converter.ts` is purely additive within the merge branch. Risk of regression on non-reasoning merges is essentially zero.
- Cast pattern `as ExtendedChatCompletionAssistantMessageParam` is used consistently with the rest of the file (the same cast appears earlier in the same function for `tool_calls`). Style-consistent.

## Risk

Low. The fix is local to `mergeConsecutiveAssistantMessages` (one branch, ~10 lines of net logic), the new behaviour only fires when the merged result has reasoning content, and the test surface is six well-targeted cases including the realistic orphaned-tool-call trigger pattern. Backward compatible by construction.

## Verdict

**merge-as-is** — clean, surgical fix for a real cross-provider correctness bug, with both the in-isolation case and the realistic production trigger pattern tested. No nits worth holding the merge for.
