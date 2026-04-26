---
pr: 3637
repo: QwenLM/qwen-code
sha: b431ec234101665e7e5cdb57f7ded02e59d9521a
verdict: merge-after-nits
date: 2026-04-26
---

# QwenLM/qwen-code #3637 — fix(core): preserve reasoning_content when merging consecutive assistant messages (#3619)

- **Author**: wenshao
- **Head SHA**: b431ec234101665e7e5cdb57f7ded02e59d9521a
- **Size**: +163/-0 across 2 files (`packages/core/src/core/openaiContentGenerator/converter.ts`, `converter.test.ts`)
- Fixes: #3619

## Scope

Closes a silent-drop bug in `mergeConsecutiveAssistantMessages` (`packages/core/src/core/openaiContentGenerator/converter.ts:1321-1380`). The existing function correctly merged `content` and `tool_calls` across two adjacent assistant turns, but `reasoning_content` from the second message was silently discarded. When DeepSeek thinking mode is active, the next request lands at the API missing the reasoning that DeepSeek expects to round-trip, and the API returns:

> HTTP 400: The reasoning_content in the thinking mode must be passed back to the API.

This matches the user-reported "chats fine at first, breaks after a while" symptom in #3619 — the merge typically fires only after `cleanOrphanedToolCalls` removes the tool message that was sitting between two assistant turns, which becomes statistically more likely the longer a session runs.

## Specific findings

- **Fix mirrors existing merge logic precisely** (`converter.ts:1326-1342`):
  ```ts
  const lastReasoning =
    'reasoning_content' in lastMessage
      ? ((lastMessage as ExtendedChatCompletionAssistantMessageParam).reasoning_content ?? '')
      : '';
  const currentReasoning = ... ;
  const combinedReasoning = [lastReasoning, currentReasoning].filter(Boolean).join('');
  // ...
  if (combinedReasoning) {
    (lastMessage as ExtendedChatCompletionAssistantMessageParam).reasoning_content = combinedReasoning;
  }
  ```
  The shape is identical to how `tool_calls` is merged immediately above. Same separator policy as `content` (empty string), same null-defensive `?? ''` (handles both missing key and explicit `null`), same write-only-if-non-empty guard. Reads cleanly.

- **Test coverage is the high-value kind, not the box-checking kind.** Four tests at `converter.test.ts:2022-2148`:
  1. *Both sides have reasoning* — asserts `reasoning_content === 'thinking step 1thinking step 2'`.
  2. *Only second side has reasoning* — explicit comment in the test calls out that this is "the highest-value regression: previously the second message's reasoning_content was silently dropped during merge, leaving the resulting assistant turn with no reasoning_content at all — exactly what triggers DeepSeek thinking mode 400". This is the exact failure mode #3619 reported and the test pins it.
  3. *Only first side has reasoning* — asserts the existing single-side reasoning isn't accidentally clobbered.
  4. *Three consecutive assistant messages* — asserts the fold-left is correct (`r1` + (no r2) + `r3` = `r1r3`).

  Test #2 is the regression test that matters; the other three triangulate the merge contract. This is exactly the test shape I'd ask for.

- **Author's "honest scope note" is exemplary.** From the PR body:
  > This is a **speculative fix based on code audit** — the path matches the reported symptom and the dropped-reasoning bug is real, but I have no direct repro from a failing user session. The data flow audit (response → history → next request) preserves `reasoning_content` everywhere except this merge, so this is the most likely remaining culprit. If a user with #3619 still reproduces after this lands, we have a clean signal that the root cause is elsewhere.

  This is the right way to land a fix when you can't repro: state explicitly that the patch is speculative-but-data-flow-justified, and articulate the disconfirming signal (#3619 still firing). That preserves the option to keep digging without needing to revert.

- **Independent of the #3304/#3579 conflict.** Author explicitly notes this doesn't require a maintainer policy decision on the model-switch strip. Reviewers can land this in isolation.

- **Nit: separator policy should be a comment, not just code.** The merge uses `[lastReasoning, currentReasoning].filter(Boolean).join('')` — empty string. That matches `content`'s merge, which is right because reasoning is a continuous monologue not two paragraphs. But the next contributor may try to "fix" this to `'\n'` thinking it's a bug. One line above the `.join('')` would prevent the round-trip: `// Empty separator: matches content/tool_calls merge convention; reasoning is a continuous stream, not paragraphs.`

- **Nit: `'reasoning_content' in lastMessage` is correct but redundantly verbose.** `(lastMessage as ExtendedChatCompletionAssistantMessageParam).reasoning_content ?? ''` would have produced the same result with one fewer narrowing check, since `?? ''` already handles undefined. Not worth blocking on; the explicit `in` check has slightly better intent-signaling.

- **Nit: the speculative-fix flag in the comment is good — keep it.** The block comment at `converter.ts:1325-1330` explains the failure mode and the trigger. Don't let a future maintainer prune that to "// merge reasoning" — the *why* is the whole point of this comment.

## Risk

Low. The merge gate (`lastMessage.role === 'assistant' && currentMessage.role === 'assistant'`) is unchanged. The new code path only writes `reasoning_content` when the combined value is truthy, so messages that never had reasoning are unaffected. Worst-case mis-fire: two assistant messages that shouldn't have been merged get their reasoning concatenated — but the existing `content`/`tool_calls` merge has the same exposure, so this PR doesn't expand the blast radius.

If the fix doesn't resolve #3619, the author has already framed the disconfirming signal: a user reporting the same `reasoning_content must be passed back` 400 after this lands means root-cause is elsewhere (response parsing, history persistence, or the next hop in the request pipeline). That's a productive next step, not a regression.

## Verdict

**merge-after-nits** — the patch is the right shape, the tests are the right shape, and the honest scope note is the right shape. Two trivial nits to keep the next contributor from breaking the contract:
1. Add a one-line comment explaining the empty-string separator convention above the `.join('')`.
2. Optionally collapse the `'reasoning_content' in X` checks to direct property access with `?? ''` — *or* leave as-is for explicit intent. Reviewer's call.

## What I learned

When the merge function for a composite type silently drops one of its component fields, the bug is invisible until something downstream specifically *requires* that field. DeepSeek's thinking mode is one such consumer. The lesson generalizes: any time you write a `mergeXY` function over a record type, your test matrix should include "only the second value is set" for *every* field, not just the obvious string/list ones. This PR's test #2 is the model — it explicitly names which field, which side, and which downstream consumer breaks. That's the test you write when you respect the next person who'll touch this code.
