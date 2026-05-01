# sst/opencode#25245 — feat(processor): add plugin stream hooks for tool streaming lifecycle

- **Author:** qz1543706741
- **Head SHA:** `6073c1c331b17736c63910d8b22466a8335b7094`
- **Base:** `dev`
- **Size:** +190 / -28 across 2 files
- **Files changed:** `packages/opencode/src/session/processor.ts`, `packages/plugin/src/index.ts`

## Summary

Introduces three new plugin hooks (`tool.stream.start`, `tool.stream.delta`, `tool.stream.resolve`) that let plugins intercept how streaming tool calls are *displayed* without changing actual execution. Motivated by the bash-routing use case where a plugin wants to translate `bash(cmd)` into a more readable shape but currently can only either parse a partial JSON stream itself or wait until execute time. As a side effect, this PR also reworks the doom-loop detector to address the same scope+filter bugs flagged separately in #25242.

## Specific code references

- `packages/plugin/src/index.ts:282-311`: three new hook signatures. `tool.stream.start` returns `{ defer: boolean }` so a plugin can opt out of immediate `ToolPart` creation; `tool.stream.delta` accumulates `raw` and resolves `input?` / `display?` once JSON is parseable; `tool.stream.resolve` rewrites `tool`/`args` and attaches `metadata` for the final part. The output shape is well-typed — `display.metadata` is the operator-visible attach-point for plugin annotations.
- `processor.ts:8-19`: `ToolCall` shape change makes `partID`/`messageID`/`sessionID` optional and adds `toolName`/`raw`/`providerExecuted`. The optionals are load-bearing — deferred calls have no part yet — and `readToolCall` at `:46` correctly guards `if (!call.partID || !call.messageID || !call.sessionID) return` so the existing `getPart` consumers don't crash on a deferred entry.
- `processor.ts:22-23,31`: new `deferredCallIDs: Set<string>` on `ProcessorContext`, with cleanup in `settleToolCall` at `:39`. This is the gate that distinguishes "deferred, drive from delta" from "stamped at start, ignore deltas" — the `tool-input-delta` case at `:122` fast-paths `if (!ctx.deferredCallIDs.has(value.id)) return` so non-deferred calls keep the existing zero-cost behavior.
- `processor.ts:54-58`: `mergeToolMetadata(...values)` collapses provider metadata + `providerExecuted` flag + plugin-resolved metadata into one merged object, returning `undefined` when all sources are empty so the `metadata?` field on `ToolPart` stays optional rather than becoming `{}`. Used at `:145-148`, `:163-165`, `:206-210`, `:225-229` — every call site that used to do bespoke conditional spreading.
- `processor.ts:60-77`: new `recentMatchingToolParts` widens doom-loop scope from "current message only" to "all messages since last user turn" via `MessageV2.filterCompactedEffect(ctx.sessionID)` + `findLastIndex(role === "user") + 1`, and inverts the slice/filter order — filter by `tool + args` identity *first*, then slice the last `DOOM_LOOP_THRESHOLD`. The old code at the now-deleted `:243-258` did `parts.slice(-DOOM_LOOP_THRESHOLD).every(...)` which silently passed the moment a brief reasoning part appeared in the tail window — exactly the doom-loop shape the detector exists to catch. This consolidates with #25242 (the PR body explicitly calls it a supersede).
- `processor.ts:99-102`: deferred-call early return after `tool.stream.start` correctly happens *before* `session.updatePart`, so a deferred plugin can suppress the placeholder part entirely. The `ctx.toolcalls[value.id]` is still pre-populated at `:87-93` so subsequent deltas have `toolName`/`raw` to read.
- `processor.ts:212-241`: fallback `ToolPart` creation when `tool-call` arrives with no prior `tool-input-start` (the provider-executed path). Construction is symmetric with the streaming path — same `mergeToolMetadata` call, same `partID`/`messageID`/`sessionID` capture into `ctx.toolcalls`. The `current?.done ?? (yield* Deferred.make<void>())` keeps the lifetime contract intact for callers awaiting completion.

## Verdict

**merge-after-nits**

Excellent design — the three-hook surface (`start`/`delta`/`resolve`) is a clean staging-out of what was previously locked inside the processor, and the fact that this PR fixes the same doom-loop scope+filter bug as #25242 in a more comprehensive way (now applied across both the streaming and non-streaming paths) is a real win. Nits: (a) the `tool.stream.delta` output type at `plugin/src/index.ts:298-302` describes `raw`/`input`/`display` but doesn't document the contract that returning `input` for the first time is what triggers the lazy `ToolPart` creation at `processor.ts:133-156` — worth a JSDoc line so plugin authors don't accidentally return `input` on every delta and create N parts; (b) the `JSON.stringify(part.state.input) === JSON.stringify(input.args)` identity check in `recentMatchingToolParts` at `:74` is sensitive to key ordering for deeply nested objects — fine for the JSON-from-LLM case where ordering is stable, but worth a comment naming the assumption; (c) needs a coordination note with #25242 about which one lands first since both touch the same doom-loop block.
