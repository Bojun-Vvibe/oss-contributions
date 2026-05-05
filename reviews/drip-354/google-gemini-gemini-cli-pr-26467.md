# google-gemini/gemini-cli PR #26467 — feat: allow queuing messages during compression

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26467
- Head SHA: `f6dbf52ac1e5b705cac51134baf8871c2b41a74f`
- Size: +100 / -12 across AppContainer, compressCommand,
  useGeminiStream, plus context plumbing and tests
- Linked issue: gemini-cli#24071

## Summary

Threads a new `isCompressing: boolean` UI state from
`AppContainer` (`packages/cli/src/ui/AppContainer.tsx:268`) all the
way through to `useGeminiStream`. The `/compress` slash command in
`packages/cli/src/ui/commands/compressCommand.ts:40,82` flips it
true on entry and false in `finally`. The streaming-state
calculator in
`packages/cli/src/ui/hooks/useGeminiStream.ts:172-205` now treats
`isCompressing` as if the model were responding, so the input area
correctly shows "agent is busy" UI while compression runs.

The behavioral hook is in `AppContainer.tsx:1397-1402`:

```ts
const isAgentRunning =
  (streamingState === StreamingState.Responding && !isCompressing) ||
  isToolExecuting(pendingHistoryItems);
```

This carve-out means: when the agent is "Responding" *because* of
compression, follow-up messages should NOT be treated as live
steering hints — they should be queued until compression finishes.
And in `AppContainer.tsx:1504-1507`:

```ts
const isInputActive =
  !initError &&
  (!isProcessing || isCompressing) &&
  …
```

…input stays active during compression so the user can type and
have it queued.

## What I like

- The plumbing is consistent: `setIsCompressing` is added to
  `UIActions` (`UIActionsContext.tsx:100`), `isCompressing` to
  `UIState` (`UIStateContext.tsx:189`), `CommandContext.ui`
  (`commands/types.ts:96`), and the slash-command processor
  (`slashCommandProcessor.ts:92,251`). No half-wired surfaces.
- The compress command uses `try { setIsCompressing(true); …
  } finally { setPendingItem(null); setIsCompressing(false); }`
  (`compressCommand.ts:40-83`). The flag is *guaranteed* to clear
  even on throw — important because a stuck `isCompressing=true`
  would silently swallow every subsequent user message into the
  queue forever.
- New integration test in
  `AppContainer.test.tsx:3596-3625` actually wires up the *real*
  `useMessageQueue` (via `vi.importActual`, line 3604-3608), flips
  `setIsCompressing(true)`, submits a message, and asserts
  `messageQueue` contains it. That's a much stronger test than
  asserting on a mocked queue's `addMessage` call.
- Three new `compressCommand.test.ts` assertions
  (lines 66, 96, 110, 129, 145) lock in: (a) flag is set true
  before any work, (b) flag is set false after success, (c) flag is
  set false in the rejected/thrown branches. Coverage of all four
  finally-block branches.

## Concerns

1. **`isAgentRunning` carve-out is subtle.** Reading
   `(streamingState === Responding && !isCompressing)` you have to
   hold in your head that the streaming state is *itself* set to
   `Responding` *because* of `isCompressing`
   (`useGeminiStream.ts:204`). So the guard is essentially
   "Responding-but-not-because-of-compression". A small named
   helper like `isAgentRespondingToUser(streamingState,
   isCompressing)` or even a comment on the line would dramatically
   improve readability.

2. **`isInputActive` change is broader than needed.**
   `(!isProcessing || isCompressing)` (line 1506) opens the input
   any time `isCompressing` is true, regardless of `isProcessing`.
   If `isProcessing` were true *and* `isCompressing` were true
   (e.g. compression triggered mid-tool-execution by another
   surface), the input would now be active. Today
   `compressCommand.ts` doesn't mix those states, but the contract
   is loose — worth a comment or an explicit check.

3. **Test mocking signature drift.** `useGeminiStream` now takes a
   19th positional arg `isCompressing` (`useGeminiStream.test.tsx:
   504-506`). The test does `undefined, undefined, props.isCompressing`
   to pad to position 19. Positional 19-arg callsites are a code
   smell — they suggest this hook should be taking a typed options
   object. Out of scope for this PR but worth a follow-up.

4. **`mockedUseGeminiStream.mockImplementation((...args) => { const
   isCompressing = args[19]; … })`** in
   `AppContainer.test.tsx:415-421` indexes positionally into
   `args[19]`. If anyone adds another arg upstream of position 19,
   this test silently asserts on the wrong field. A typed mock
   helper would prevent the bug class.

5. **No test for the long-running compression UX.** What happens
   if the user queues 5 messages during a 30-second compression?
   The PR's compressCommand flips the flag on/off, but I don't see
   a test that asserts the queued messages are actually *flushed*
   (sent to the model) once `isCompressing` becomes false. That's
   the whole user-facing point of issue #24071, and it relies on
   `popAllMessages` (added in the test mock at lines 442-447) being
   called by the streaming hook on the right state transition.
   Worth one more end-to-end test.

## Verdict

**merge-after-nits** — the design is right (compression should
behave like "model is busy" for streaming-state purposes but
should NOT route input to steering hints), the plumbing is
consistent, and the four-branch finally-block coverage on
`setIsCompressing(false)` is exactly what I'd ask for. Tighten the
`isAgentRunning` readability (#1) and add the queue-flush
end-to-end test (#5). The 19-positional-arg hook signature is
technical debt to track separately.
