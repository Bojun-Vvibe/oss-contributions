# sst/opencode #25901 — fix(acp): return stopReason: cancelled (not end_turn) on user cancel

- **PR:** sst/opencode#25901
- **Head SHA:** `f9502791b16dd77e7488c867352834a0579b3e09`
- **Files:** 2 changed (+117 / -4)

## What changed

- `packages/opencode/src/acp/agent.ts:1471-1488` introduces a local `wasCancelled(msg)` helper that detects user cancellation by checking `msg?.error?.name === "MessageAbortedError"` (the tag set by `session/processor.ts` halt → `MessageV2.fromError`). The comment correctly explains why the existing `session/prompt` path can't tell the two cases apart on its own — the RPC resolves with the assistant message in both the natural-finish and the `session/cancel` interrupt paths.
- Both `prompt()` return sites (`agent.ts:1503-1508` for the `!cmd` plain-prompt branch and `agent.ts:1525-1530` for the slash-command branch) now return `stopReason: "cancelled"` in place of the unconditional `"end_turn"` when the message was aborted, and **omit `usage` entirely** on cancel rather than emitting `buildUsage(msg)` over the zero-initialised `msg.tokens`.
- New regression tests at `packages/opencode/test/acp/event-subscription.test.ts:727-822` cover both branches: one fakes `sdk.session.prompt` to resolve with an assistant message tagged `error.name = "MessageAbortedError"` and asserts `result.stopReason === "cancelled"` and `result.usage === undefined`; the other validates the unchanged happy-path (`stopReason: "end_turn"`, `usage.totalTokens === 350`).

## Risks / notes

- The omit-`usage`-on-cancel decision is well-reasoned (provider has charged for prompt tokens but the SDK never emitted `finish-step` so the client-side count is genuinely unknown), but it does represent a behaviour change for any ACP client that was previously parsing the `totalTokens: 0` it was getting back. The PR comment makes the rationale clear; worth a release note.
- The fix only patches the two `return` sites inside `prompt()`. If a future code path adds a third return that calls `buildUsage(msg)` unconditionally, the same bug will silently regress — minor but a `getStopReasonAndUsage(msg)` helper would have made that harder to undo. Not a blocker.
- Cancellation detection relies on the string literal `"MessageAbortedError"` matching the producer in `session/processor.ts`. There's no shared constant; if either side renames the error class the check silently degrades to `end_turn`. A shared symbol would be safer.

## Verdict

**merge-after-nits** — the behaviour fix is correct and the tests pin both the cancel and non-cancel paths. Worth extracting a tiny shared `MESSAGE_ABORTED_ERROR_NAME` constant and adding a release note about `usage` being omitted on cancel before merge, but neither is load-bearing.
