# sst/opencode PR #25245 — feat(processor): add plugin stream hooks for tool streaming lifecycle

- Link: https://github.com/sst/opencode/pull/25245
- SHA: `e6b9974a5ef64f75c6f74a8cffb6803b819f861d`
- Author: qz1543706741
- Stats: +192 / −28, 2 files

## Summary

Adds three plugin lifecycle hooks (`tool.stream.start`, `tool.stream.delta`, `tool.stream.end`) so plugins can intercept how tool calls are surfaced to the UI without altering execution. The bulk of the change is in `packages/opencode/src/session/processor.ts`, with the matching public type surface in `packages/plugin/src/index.ts`.

The `ToolCall` shape changes meaningfully: `partID`, `messageID`, `sessionID` become optional, and new fields `toolName`, `raw`, and `providerExecuted` are introduced. A parallel `deferredCallIDs: Set<string>` is added to `ProcessorContext`. The `readToolCall` helper now bails when any of the three id fields are missing — that bailout (`if (!call.partID || !call.messageID || !call.sessionID) return`) is the right defensive shape but it silently swallows what was previously an invariant.

## Specific references

- `packages/opencode/src/session/processor.ts` L60–L67: making the three identifier fields optional weakens an invariant that several call-sites used to rely on. Worth a code comment explaining when/why a `ToolCall` legitimately exists without a `partID` (presumably during the new pre-message-allocation streaming window).
- `packages/opencode/src/session/processor.ts` L140–L143: `settleToolCall` now also removes from `deferredCallIDs`. Good — both sets stay consistent. Confirm there is no other path that inserts into `deferredCallIDs` without going through `settleToolCall` on cleanup.
- `packages/opencode/src/session/processor.ts` L150–L157: `readToolCall` early-returns silently when any id is missing. Consider logging at debug so plugin authors can diagnose hooks that fire too early.
- `packages/opencode/src/session/processor.ts` L162–L186: `recentMatchingToolParts` does `JSON.stringify(part.state.input) === JSON.stringify(input.args)` for de-dup. The comment claims key order is stable from provider streams; that is fragile if any plugin or normaliser ever re-keys the args object. A canonical-stringify or a structural comparator would be safer.
- `packages/plugin/src/index.ts`: confirm the new hook type surface is additive and doesn't break existing plugin signatures (couldn't see breaking renames in the diff).

## Verdict

verdict: merge-after-nits

## Reasoning

The hooks are a useful extension point and the wiring through `deferredCallIDs` and `settleToolCall` looks consistent. The two real concerns are (1) loosening `ToolCall` identifier invariants without a comment explaining the new lifecycle window, and (2) the `JSON.stringify` equality used in `recentMatchingToolParts` for the doom-loop guard. Both are fixable without architectural changes.
