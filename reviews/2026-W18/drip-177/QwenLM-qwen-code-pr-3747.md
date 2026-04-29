# QwenLM/qwen-code#3747 — fix(core): replay DeepSeek reasoning_content on all assistant turns

- **Repo:** QwenLM/qwen-code
- **PR:** [#3747](https://github.com/QwenLM/qwen-code/pull/3747)
- **Head SHA:** `0d5bf1181f1a7f103b335bfa57cee44f204e8736`
- **Author:** tanzhenxin
- **Size:** +9 / -12 across 2 files

## Summary

Extends the prior `reasoning_content` normalization fix (#3729) to apply to
*every* prior assistant turn in the request, not just turns that carry
`tool_calls`. Live testing against `api.deepseek.com` shows the API also
rejects follow-up requests when an assistant turn *without* `tool_calls`
is missing `reasoning_content` — same HTTP 400 (`The reasoning_content in
the thinking mode must be ...`).

## What's actually going on

The prior fix in #3729 added `ensureReasoningContentOnToolCalls` —
self-described as "DeepSeek's thinking mode requires reasoning_content to
be replayed on every prior assistant turn that carried tool_calls." The
function bailed early on assistant turns without `tool_calls`:

```ts
if (!Array.isArray(message.tool_calls) || message.tool_calls.length === 0) {
  return message;
}
```

This PR removes that early-return at
`packages/core/src/core/openaiContentGenerator/provider/deepseek.ts:120-122`
(deleted lines), so the function now always reaches the
`extended.reasoning_content === '' || ... === undefined` check and stamps
empty-string when missing. The function is no longer well-named — it's
described as `ensureReasoningContentOnToolCalls` but now applies to *all*
assistant turns. Worth renaming to `ensureReasoningContentOnAssistantTurns`
in the same PR — it's a small semantic drift now and a bigger maintenance
trap later when someone reads the helper name and assumes it only touches
tool-calling turns.

The comment block at `deepseek.ts:108-115` is updated to reflect the new
scope (`...on every prior assistant turn, including ones without tool_calls`).

## Specific line refs

- `packages/core/src/core/openaiContentGenerator/provider/deepseek.ts:108-115`
  — comment block updated. Now reads correctly: "DeepSeek's thinking mode
  requires reasoning_content to be replayed on every prior assistant turn,
  including ones without tool_calls."
- `packages/core/src/core/openaiContentGenerator/provider/deepseek.ts:120-122`
  (deleted) — the early-return on `!Array.isArray(message.tool_calls) ||
  message.tool_calls.length === 0`. Removing this is the substantive
  behavior change.
- `packages/core/src/core/openaiContentGenerator/provider/deepseek.test.ts:182-185`
  — comment update on the existing tool-calling test, now noting "any prior
  assistant turn" instead of "tool-calling assistant turn."
- `packages/core/src/core/openaiContentGenerator/provider/deepseek.test.ts:248`
  — test renamed from `does not add reasoning_content to assistant turns
  without tool_calls` to `injects empty reasoning_content on assistant turns
  without tool_calls` — captures the inverted intent.
- `packages/core/src/core/openaiContentGenerator/provider/deepseek.test.ts:262`
  — assertion flipped from `expect(assistant.reasoning_content).toBeUndefined()`
  to `expect(assistant.reasoning_content).toBe('')`. This is the right
  test — it's the exact bug-shape that was producing the API 400.

## Reasoning

The change is correct and minimal. The previous fix #3729 made an
underspecified assumption — "the API only complains about tool-calling
turns" — and live testing rejected that assumption. The new behavior
matches what the API actually demands.

Three things worth flagging:

1. **Function name now misleads.** `ensureReasoningContentOnToolCalls` is
   the helper's exported name and it no longer only touches tool-calling
   turns. Rename to `ensureReasoningContentOnAssistantTurns` (or just
   `ensureReasoningContent`) in the same PR — names that lie are worse
   than names that are wordy. Single call site presumably, so the rename
   is safe; if it's exported across the module boundary, add a short alias
   to avoid breaking imports.

2. **Behavior change is broader than the PR title suggests.** The old
   helper returned the message unchanged for non-tool-calling assistant
   turns; the new one *mutates* (well, returns a new object with) the
   `reasoning_content` field stamped to `''`. Any downstream consumer that
   reads back the request payload and expected
   `assistant.reasoning_content === undefined` for non-tool turns now sees
   `''` instead. Probably no such consumer exists — this is a
   request-builder, not a request-cache — but worth a mental check.

3. **No upstream-API regression test.** The fix was driven by live API
   behavior; the test suite still uses synthetic message arrays. A nightly
   integration test against `api.deepseek.com` (gated on the API key
   secret) would catch the next "DeepSeek tightened the contract again"
   change without requiring a user to file an issue. Out of scope for this
   PR, but worth a follow-up tracking issue — this is the second time the
   exact same class of bug has hit (#3729 was the first, #3747 is the
   second), and there's no reason to assume DeepSeek's thinking-mode
   contract is now stable.

The diff is otherwise impossible to make smaller. Three lines deleted from
the function, two comment blocks updated, one test renamed and re-asserted.

## Verdict

**merge-after-nits** — rename `ensureReasoningContentOnToolCalls` →
`ensureReasoningContentOnAssistantTurns` (or `ensureReasoningContent`)
in the same PR since the helper no longer only touches tool-calling turns
and the misleading name will cost more in future debug time than the
rename costs now; file a follow-up tracking issue for a nightly
integration test against `api.deepseek.com` (gated on the API-key secret)
since this is the second time DeepSeek's `reasoning_content` contract has
silently tightened (#3729 → #3747) and the next tightening shouldn't
require a user to file an issue; and add a one-line note in the PR body /
CHANGELOG that non-tool-calling assistant turns now carry an empty-string
`reasoning_content` field in the outgoing payload — downstream consumers
that read back the request payload may need to update their expectations.
