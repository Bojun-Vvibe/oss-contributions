# sst/opencode #25242 — fix(processor): doom loop detection scope and filter order

- **PR**: https://github.com/sst/opencode/pull/25242
- **Head SHA**: `5ea7af24e16613e5871b424d3f4b20ecec4355f4`
- **Files reviewed**: `packages/opencode/src/session/processor.ts`
- **Date**: 2026-05-01 (drip-230)

## Context

The session processor's "doom loop" detection is meant to stop a model that keeps invoking
the same tool with the same args N times in a row. The PR closes two real-world holes
where the detector silently failed.

## Bug 1 — wrong scope

`processor.ts` previously called `MessageV2.parts(ctx.assistantMessage.id)`, which
returns parts from **the current assistant message only**. Most agent loops produce one
assistant message per LLM call, so a model that does the same `read_file` three turns in a
row never accumulates `DOOM_LOOP_THRESHOLD` matching parts inside any one message — each
message has 1 matching part and the threshold (presumably 3+) is never met. The detector
fires only on the pathological case where the model emits N identical tool calls inside a
*single* assistant turn, which is not the common doom-loop shape.

Fix at `processor.ts:152-170` switches to
`MessageV2.filterCompactedEffect(ctx.sessionID)` and slices from
`messages.findLastIndex(m => m.info.role === "user") + 1` onward, i.e. all assistant
messages since the most recent user turn. That's the correct scope: a "loop" is bounded by
the user-input boundary, not by an LLM call boundary.

## Bug 2 — slice-then-filter inverts the predicate

The old code at the now-deleted `processor.ts:305-320`:

```ts
const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)
if (
  recentParts.length !== DOOM_LOOP_THRESHOLD ||
  !recentParts.every(part => part.type === "tool" && part.tool === value.toolName && ...)
) return
```

Slice-first means the tail-N window may contain text/reasoning parts. `every(...)` over
that window then returns `false` whenever a non-tool part appears in the tail — which
makes the detector pass-through (return early via the `!...every(...)` branch) on
**exactly the case it's supposed to catch**: a model that interleaves brief reasoning
text between identical tool calls.

The new helper at `:152-170` reverses the order — `filter` first by tool+args identity,
then `.slice(-DOOM_LOOP_THRESHOLD)`:

```ts
return messages
  .slice(lastUserIndex + 1)
  .flatMap((msg) => msg.parts)
  .filter(
    (part): part is MessageV2.ToolPart =>
      part.type === "tool" &&
      part.tool === input.tool &&
      part.state.status !== "pending" &&
      JSON.stringify(part.state.input) === JSON.stringify(input.args),
  )
  .slice(-DOOM_LOOP_THRESHOLD)
```

and the call site at `:324` collapses to a length check only:

```diff
- const parts = MessageV2.parts(ctx.assistantMessage.id)
- const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)
- if (recentParts.length !== DOOM_LOOP_THRESHOLD || !recentParts.every(...)) return
+ const recentParts = yield* recentMatchingToolParts({ tool: value.toolName, args: value.input ?? {} })
+ if (recentParts.length !== DOOM_LOOP_THRESHOLD) return
```

That's the correct invariant: the helper returns *only* matching tool parts, so
`length === DOOM_LOOP_THRESHOLD` is the predicate.

## Observations

1. **`JSON.stringify` for arg-equality is the same key as before** — same hazards (key
   ordering, undefined→missing collapse, NaN/Infinity), but unchanged from the old code
   at the deleted call site. Not a regression.

2. **`pending` filter preserved.** The new helper keeps `part.state.status !== "pending"`
   so the in-flight tool call that *triggered* the check is excluded from its own
   denominator — important so the threshold counts completed prior calls, not the current
   one being decided.

3. **`value.input ?? {}` is the right defensive default.** A tool call with no args
   (`value.input === undefined`) now compares against `JSON.stringify({}) === "{}"`. Two
   no-arg calls will correctly match each other (`"{}" === "{}"`), and a no-arg call
   won't accidentally match `JSON.stringify(undefined) === undefined` (string vs
   undefined comparison would be false-y).

4. **`Effect.fn("SessionProcessor.recentMatchingToolParts")` traces correctly.** The
   helper is wrapped in `Effect.fn(...)` with a trace name matching the surrounding
   convention, so it shows up as its own span in the processor trace tree.

## Nits

- A `findLastIndex(... === "user")` returning `-1` (no prior user turn in the session,
  which can happen for the very first message) makes `messages.slice(0)` — i.e. all
  messages. That's fine for the current contract but worth a one-line comment so the
  next reader doesn't think it's a bug.
- Repeated `JSON.stringify(part.state.input)` per filter call is O(N) extra serializations
  per check; pre-computing `const argsKey = JSON.stringify(input.args)` once outside the
  filter would tighten the hot path.

## Verdict

merge-after-nits
