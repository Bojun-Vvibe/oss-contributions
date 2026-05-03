# Review: sst/opencode #25255 — fix(processor): fix doom loop detection scope and filter order

- PR: https://github.com/sst/opencode/pull/25255
- Head SHA: `5ea7af24e16613e5871b424d3f4b20ecec4355f4`
- Author: qz1543706741
- Size: +21 / -14

## Summary

Two coupled bugs in the doom-loop detector at
`packages/opencode/src/session/processor.ts`. (1) Scope: the existing
code called `MessageV2.parts(ctx.assistantMessage.id)` which only
returns parts from the *current* assistant message, so a model that
makes the same tool call across three separate turns never trips the
detector. (2) Filter order / accuracy: the previous logic took the
last `DOOM_LOOP_THRESHOLD` parts of *any* type and then required them
to all be matching tool calls — so a single intervening text part
falsely "broke" the streak. The PR replaces both with a new helper
`recentMatchingToolParts` that walks back to the last user message,
flattens parts across messages, *filters first* on tool/args, then
slices the tail.

## Specific citations

- `packages/opencode/src/session/processor.ts:152-170` — the new
  `recentMatchingToolParts` Effect.fn. It uses
  `MessageV2.filterCompactedEffect(ctx.sessionID)` (correctly:
  compacted history is the right window — pre-compaction parts are
  already summarised and shouldn't count toward a "loop") then
  `findLastIndex(... role === "user")` to bound the lookback, then
  `flatMap` + filter + tail slice. The filter predicate matches what
  the old inline filter checked: `type === "tool"`,
  `tool === input.tool`, `state.status !== "pending"`,
  `JSON.stringify(input) === JSON.stringify(args)`.
- `packages/opencode/src/session/processor.ts:321-323` — at the call
  site, the old `parts.slice(-DOOM_LOOP_THRESHOLD)` + `every(...)`
  block (10 lines) collapses to one yield + one length check. Cleaner.
- The `JSON.stringify` arg comparison is unchanged from the original.
  It was a pragmatic equality check before and remains one — it will
  reject semantically equal but key-order-different objects, which is
  the normal failure mode here (the model regenerates the same call
  with the same key order, so this is fine in practice).

## Verdict

**merge-after-nits**

## Rationale

This is the right fix for both bugs. Bug 1 was a real correctness gap
— cross-turn loops were the scenario the detector most needed to
catch, and it silently didn't. Bug 2 was a UX regression where one
intervening assistant text part would reset the loop counter. The
unified helper removes both.

The "lookback bounded by last user message" choice is the key design
decision and it's the right one: a fresh user turn is the cleanest
"the user knows the loop is happening and is intervening" signal, so
resetting the streak there is correct. Pre-user-message tool calls
shouldn't combine across turns.

What's missing: zero tests. The helper is a pure-ish function over
session state — straightforward to unit-test by stubbing
`MessageV2.filterCompactedEffect`. For a detector that drives a
permission prompt, a test asserting "3 identical tool calls across 3
messages → trips" and "interleaved user message → resets" should be
table stakes.

Performance: `flatMap` over all post-last-user messages, then
`JSON.stringify` per part for arg comparison. For a session with
hundreds of parts this is fine, but if `DOOM_LOOP_THRESHOLD` is small
(typically 3) you could short-circuit by walking parts in reverse and
collecting only matches. Probably premature optimisation here.

## Nits

1. Add unit tests covering the cross-message and interleaved-text
   cases that the PR description explicitly calls out.
2. `JSON.stringify(input) === JSON.stringify(args)` — extract to a
   `sameToolArgs(a, b)` helper, both for readability and so a future
   move to a stable-key serializer or deep-equal can happen in one
   spot.
3. `messages.findLastIndex(...)` returns -1 if no user message
   exists. `slice(0)` is fine in that case (returns all messages),
   but worth a one-line comment confirming this is intentional for a
   session that starts with a system/assistant message.
