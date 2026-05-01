# google-gemini/gemini-cli #26286 — fix stale state in /rewind

- **Repo:** google-gemini/gemini-cli
- **PR:** https://github.com/google-gemini/gemini-cli/pull/26286
- **HEAD SHA:** `0cf502e72c8612acaa9ee48380ea1f04bb5aa223`
- **Author:** joshualitt (commits: `602d685` initial fix, `0cf502e` reviewer feedback)
- **Verdict:** `merge-after-nits`

## What the diff does

Five-file fix for `/rewind` leaving the chat in a stale state where the
on-disk recording diverged from the in-memory chat history (closes #25646):

1. **Replace `setHistory` with `resumeChat` in slash command processor.**
   `packages/cli/src/ui/hooks/slashCommandProcessor.ts:551-553` —
   `'load_history'` arm now `await config?.getGeminiClient()?.resumeChat(result.clientHistory)`
   instead of `config?.getGeminiClient()?.setHistory(result.clientHistory)`.
   Test mock at `slashCommandProcessor.test.tsx:580` updated:
   `setHistory: vi.fn()` → `resumeChat: vi.fn().mockResolvedValue(undefined)`.

2. **Reconstruct messages on history load.**
   `packages/core/src/services/chatRecordingService.ts:847-967` (new
   `reconstructMessagesFromHistory` private method) — walks
   `readonly Content[]`, building `MessageRecord[]` with proper
   user/gemini distinction, thought parsing via `parseThought`,
   tool-call extraction from `functionCall` parts, and the
   load-bearing **look-ahead** at `:917-952` that pairs `model`-role
   tool-call turns with the following `user`-role turn's
   `functionResponse` parts (mapping by `functionResponse.id` so sibling
   text/inlineData parts are preserved alongside).

3. **Sync history into recording at chat init.**
   `packages/core/src/core/geminiChat.ts:275-281` — in
   `chatRecordingService.initialize`, if `!resumedSessionData &&
   this.agentHistory.get().length > 0`, call
   `this.chatRecordingService.updateMessagesFromHistory(this.agentHistory.get())`.
   This handles the load-bearing case where `startChat(extraHistory)` is
   called for `/rewind` without a `resumedSessionData` blob — without this
   sync, the recording service stays empty even though the agent history
   has the rewound state.

4. **`updateMessagesFromHistory` reconstruct flag.**
   `packages/core/src/services/chatRecordingService.ts` — new optional
   `reconstruct?: boolean` parameter. When `reconstruct === true` (called
   from `geminiChat.ts:786` post-compaction or post-strip) the cache is
   force-reset; when `reconstruct` is unset and cache is non-empty,
   existing tool-result-update behavior is preserved.

5. **Type tightening.** `packages/core/src/core/client.ts:326/367` —
   `resumeChat`/`startChat` widen from `Content[]` to
   `readonly Content[]`. `packages/core/src/context/chatCompressionService.test.ts:198-201`
   updates the `getInitialChatHistory` mock signature accordingly.

6. **Tests** — three new tests at
   `chatRecordingService.test.ts:1240-1336`:
   - `repopulate cachedConversation.messages when updating from history if
     cache is empty (regression)` — pins the load-bearing /rewind path:
     empty cache + 3-message history → 3 messages reconstructed in cache.
   - `force reconstruction when reconstruct flag is true, even if cache
     is not empty` — pins the post-compaction force-reset contract.
   - `correctly reconstruct sibling parts (text/media) in tool response
     turns` — pins the look-ahead pairing: model-role `functionCall` turn
     followed by user-role turn with `[{text: 'Sibling text'},
     {functionResponse}, {inlineData}]` produces a single gemini message
     with one tool call whose `result` is the full 3-part array.

## Why the change is right

The bug shape is **two source-of-truth divergence** after `/rewind`:
1. The slash-command processor was calling `setHistory(result.clientHistory)`
   which replaced the in-memory `agentHistory` but left the
   `chatRecordingService.cachedConversation.messages` array carrying the
   pre-rewind tail. Subsequent UI reads of the recording showed stale
   messages.
2. Even after switching to `resumeChat`, the recording service's
   `initialize(undefined, kind)` path didn't sync the freshly-set
   history because the recording-service contract was "messages flow in
   via `recordMessage` calls during a turn." Bypassing turns (via /rewind
   or initial `extraHistory`) silently broke the invariant.

The fix shape is the right **state-machine invariant restoration**:

- `resumeChat` (already exists) is the recorded entry point that knows how
  to cooperate with the recording service. Replacing the raw `setHistory`
  call at `slashCommandProcessor.ts:553` aligns `/rewind` with the
  documented chat-resumption protocol.
- The `initialize` sync at `geminiChat.ts:278-281` plugs the
  initial-history-without-resumed-session hole — the conditional
  `!resumedSessionData && this.agentHistory.get().length > 0` is precise:
  if a session record was resumed, the recording service already
  rehydrated; if there's no resumed-session and no extra history, the
  empty-cache state is correct; only the new arm
  (no-resumed-session-but-has-extra-history) needs the sync.
- The `reconstruct` flag at `updateMessagesFromHistory` is the right
  escape hatch — the prior in-place "merge tool results into existing
  messages" behavior is preserved by default, while compaction/strip and
  /rewind paths opt into a full rebuild. Adding the flag instead of
  flipping the default avoids regressing the existing turn-based update
  callers.
- The look-ahead pairing at `:917-952` is the load-bearing detail of
  `reconstructMessagesFromHistory`. Genai history alternates `model` and
  `user` roles, but a tool-call turn's *response* lives in the *next*
  turn (the user-role turn that carries `functionResponse` parts). The
  reviewer-feedback commit `0cf502e` correctly handles the sibling-part
  case (the third test pins a `text + functionResponse + inlineData`
  shape) which matches the real-world Gemini protocol.

The `readonly Content[]` widening in `client.ts` and `geminiChat.ts` is a
correct downstream of "we now hand the same `history` array to multiple
consumers (agent, recording-service)" — making the contract explicitly
read-only prevents a future "let's mutate it in place to save an alloc"
refactor from re-introducing the divergence.

## Nits (non-blocking)

1. **`updateMessagesFromHistory(history)` second-arg default behavior is
   subtle.** At `chatRecordingService.ts:848`, the function default
   (`reconstruct` defaults to falsy) is "reconstruct only if cache is
   empty" — this is a *behavioral overload* on the same call site.
   Without an inline contract comment naming the three branches
   (no-flag-empty-cache, no-flag-nonempty-cache, flag-true), a future
   reader might assume the second arg is "force replace" boolean and
   miss the empty-cache reconstruction. A 5-line block comment at the
   function signature would be cheap insurance.

2. **`assert_not_called` analog missing in slashCommandProcessor test.**
   The test at `slashCommandProcessor.test.tsx:580` only asserts
   `resumeChat` is mocked; it doesn't assert the prior `setHistory` is
   *not* called. A `expect(mockClient.setHistory).not.toBeDefined()` or
   equivalent would make a regression where someone re-introduces
   `setHistory` alongside `resumeChat` (double-call) surface in test.

3. **`Assume success for reconstructed history` at `:894`** — the
   `status: CoreToolCallStatus.Success` assumption is the right
   pragmatic choice for displaying historical tool calls, but the
   inline comment should name *why* (we have no signal of failure for
   reconstructed turns, and showing them as failed would be
   misleading). One-line addition.

4. **`randomUUID()` for reconstructed message IDs** at `:868`/`:874`/etc
   means a `/rewind` followed by a re-reconstruct produces different
   IDs each time. If anything downstream key-stabilizes on the message
   ID (e.g., React `key=` props in a virtualized list), this could
   cause UI flicker. Worth a quick scan of `ChatRecordingService`
   consumers to confirm message ID is treated as opaque/throwaway.

## Verdict rationale

Well-shaped fix targeting a clear two-source-of-truth divergence with the
right entry-point realignment (`resumeChat` instead of `setHistory`), the
right state-machine sync at `initialize`, the right `reconstruct` opt-in
flag preserving existing semantics, and the load-bearing look-ahead
pairing for tool-response turns pinned by the sibling-parts test.
Nits are documentation/defensive-test improvements, none are blockers.

`merge-after-nits`
