# sst/opencode#25255 — fix(processor): fix doom loop detection scope and filter order

- **PR**: https://github.com/sst/opencode/pull/25255
- **Head SHA**: `5ea7af24e16613e5871b424d3f4b20ecec4355f4`
- **Size**: +21 / -14, 1 file
- **Verdict**: **merge-as-is**

## Context

The doom-loop guard in `processor.ts` (case `tool-input-delta` / `tool-input-end`) was supposed to detect when the model is calling the same tool with the same arguments `DOOM_LOOP_THRESHOLD` times in a row, then escalate via `permission.ask`. The previous implementation read `MessageV2.parts(ctx.assistantMessage.id)` and grabbed the last `THRESHOLD` parts, then required *all of them* to match the (tool, args) tuple. That had two real bugs the previous drip already flagged on the now-superseded #25242: scope was the assistant message (not the user-turn boundary, so a brief reasoning/text part inside the tail window let the loop pass silently), and the order was slice-then-filter (so a non-matching part anywhere in the tail killed the check).

This PR replaces that with a new helper.

## Design analysis

New `recentMatchingToolParts(input)` at `processor.ts:155-172`:
1. Calls `MessageV2.filterCompactedEffect(ctx.sessionID)` — operates on the whole session, after compaction filtering.
2. `findLastIndex(msg => msg.info.role === "user")` and slices from `lastUserIndex + 1` so the window is exactly **the current user turn**, no matter how many assistant messages the agent split itself across.
3. `flatMap(msg => msg.parts)` flattens the assistant turn(s).
4. **Filters first** by `(part.type === "tool", tool, state.status !== "pending", JSON.stringify(state.input) === JSON.stringify(args))`.
5. **Then** `.slice(-DOOM_LOOP_THRESHOLD)`.

Call site at `processor.ts:324`:
```ts
const recentParts = yield* recentMatchingToolParts({ tool: value.toolName, args: value.input ?? {} })
if (recentParts.length !== DOOM_LOOP_THRESHOLD) return
```

The two-line `if (recentParts.length !== DOOM_LOOP_THRESHOLD) return` collapses the 14-line `every(...)` predicate the old code needed. Correctness is now structurally guaranteed by the helper rather than re-asserted at the call site.

## What's right

- **Scope fix is load-bearing.** `MessageV2.parts(ctx.assistantMessage.id)` only saw the *current* assistant message; if the agent split itself across two assistant messages within a turn (e.g. compaction trigger, sub-agent return), the old check missed cross-message repetition. `filterCompactedEffect(sessionID)` + user-boundary slice is the right scope.
- **Filter-then-slice is the right order.** Old code: `parts.slice(-THRESHOLD)` then `every(...)` → any non-matching tail member silently killed the check. New code only ever counts matching parts, so reasoning/text parts interleaved with the loop don't suppress detection.
- **Pending exclusion preserved** (`status !== "pending"`) so an in-flight identical call doesn't double-count.
- **`value.input ?? {}` guard** matches the prior `value.input` direct compare (both `JSON.stringify(undefined)` and `JSON.stringify({})` would have matched the prior side, but normalizing here makes the helper's contract explicit).
- **Effect-fn naming** (`SessionProcessor.recentMatchingToolParts`) gives the trace surface a clean span name.

## Risks / nits

- `MessageV2.filterCompactedEffect(ctx.sessionID)` on every `tool-input-end` is one extra effect per matching tool call. Should be cheap (it's already the canonical reader for compaction-aware history), but worth a micro-bench if a future profile shows this hot.
- `findLastIndex(msg.info.role === "user")` returns `-1` if there is no user message yet (very edge case — first synthetic turn?), in which case `slice(0)` → entire session, which still works correctly.
- No new test arm pinning the new scope. Given #25242 was reverted/superseded specifically because of these bugs, a regression test that interleaves a non-matching reasoning part inside the tail and asserts the guard still fires would prevent another silent regression. Not blocking but recommended.

## Verdict

**merge-as-is** — surgical bug fix at the right boundary, preserves the previously-working contract, supersedes the broken #25242 cleanly. The missing regression test is the only gap and is small enough to land in a follow-up.
