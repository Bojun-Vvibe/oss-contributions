# PR #24200 — fix: preserve empty reasoning_content for DeepSeek V4

- URL: https://github.com/sst/opencode/pull/24200
- Author: ihoooohi
- Head SHA: `9fbb8a4d3b1671c5367fe920ba01943785fd13b1`

## Summary

Patches the three remaining call sites where empty `reasoning_text` /
`reasoning_content` is silently dropped by the OpenAI-compatible chat path.
DeepSeek V4 (and similar providers) require the empty-string reasoning field
to be echoed back verbatim across multi-turn calls; the prior code's
truthiness checks (`if reasoning_content`, `if reasoning != null && length > 0`,
`if part.text`) collapsed `""` into "field absent", which the server then
rejected. Companion to PR #24146 which fixed the outbound `transform.ts` path.

## Specific callouts

- `packages/opencode/src/provider/sdk/.../convert-to-openai-compatible-chat-messages.ts:95-100`
  — Outbound conversion changed from `if (part.text) reasoningText = part.text`
  to `reasoningText = part.text ?? ""`. Correct: previously an empty reasoning
  part would leave `reasoningText` undefined and the resulting message would
  carry no `reasoning_content` field at all, which DeepSeek V4 treats as a
  protocol violation when the prior assistant turn included one.
- `.../openai-compatible-chat-language-model.ts:227-235` (non-streaming) —
  Guard relaxed from `reasoning != null && reasoning.length > 0` to
  `reasoning != null`. The `null` check is correctly retained so genuinely
  absent reasoning fields don't get fabricated as empty content parts; only
  explicit `""` from the wire is preserved. This is the right shape.
- `.../openai-compatible-chat-language-model.ts:479-503` (streaming) — The
  important subtlety: the gate that controls `reasoning-start` emission and
  the gate that controls `reasoning-delta` emission are now **split**.
  - `if ("reasoning_text" in delta)` — opens the reasoning stream as soon as
    the field is present, even if its value is `""`. This ensures the
    consumer sees a `reasoning-start` matching what the provider sent, so
    the eventual `reasoning-end`/`finish` symmetry holds.
  - `if (reasoningContent)` (truthy) — only emits a delta when there's
    actual text. Avoids spamming empty deltas downstream.

  This split is exactly the right move; the only failure mode I can think of
  is a malformed provider that sends `reasoning_text: null` *as a property*
  (not just absent). `"reasoning_text" in delta` is `true` for that case, but
  the subsequent `delta.reasoning_text` is `null`, so the truthy check on
  `reasoningContent` correctly skips the delta and the start-event still
  fires. Worth a one-line comment that this split is intentional, otherwise a
  future cleanup PR will "simplify" it back into one branch.
- No tests added. This is a behavioral fix targeting a real bug (Issue #24188)
  with a narrow and easily-mockable contract — please add at least one
  fixture test that feeds a streaming response containing
  `delta.reasoning_text === ""` and asserts both `reasoning-start` and
  `finish` fire, with no `reasoning-delta` between them. Without it, the
  next refactor of this gate has nothing to anchor against.
- Does the fix compose with PR #24146 cleanly, i.e. is there any path where
  both layers now redundantly emit empty reasoning? Brief sanity check
  against the transform layer would close the loop.

## Risks

- Behavior change for any provider that previously relied on the truthy
  check to collapse spurious `reasoning_text: ""` events — those will now
  produce a `reasoning-start` … `reasoning-end` pair with no content.
  Downstream renderers that assume reasoning blocks are non-empty will
  display empty thinking sections. Low-risk, but worth a quick audit of
  the TUI reasoning renderer.

## Verdict

**Verdict:** merge-after-nits

Add a streaming-path fixture test for `delta.reasoning_text === ""` and a
short comment on the deliberate split between the `in delta` gate and the
truthy `reasoningContent` gate, then ship. The fix itself is correct.
