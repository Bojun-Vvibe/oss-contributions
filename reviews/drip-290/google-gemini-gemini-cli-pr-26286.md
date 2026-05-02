# google-gemini/gemini-cli PR #26286 — fix stale state in /rewind

- Head SHA: `0cf502e72c8612acaa9ee48380ea1f04bb5aa223`
- URL: https://github.com/google-gemini/gemini-cli/pull/26286
- Size: ~210 / -10, 6 files
- Verdict: **merge-after-nits**

## What changes

Two related fixes for `/rewind` (and adjacent `/chat resume` /
`load_history` flow) leaving the chat recording cache out of sync
with the actual `GeminiChat.history`:

1. **`load_history` now calls `resumeChat` instead of `setHistory`.**
   `slashCommandProcessor.ts:551-553` switches to
   `await config?.getGeminiClient()?.resumeChat(result.clientHistory)`,
   which goes through `GeminiChat.initialize` and so picks up the
   recording-service sync (see fix #2). The matching test update
   (slashCommandProcessor.test.tsx:580) swaps `setHistory` for
   `resumeChat: vi.fn().mockResolvedValue(undefined)`.
2. **`ChatRecordingService` reconstructs messages from history when its
   cache is empty *or* when forced.** `chatRecordingService.ts:863-887`
   adds a new `reconstructMessagesFromHistory(history)` private method
   plus a `reconstruct = false` flag on `updateMessagesFromHistory`
   (chatRecordingService.ts:893-905). The reconstruction walks the
   `Content[]` array, builds `MessageRecord` entries (user / gemini /
   tool calls / thoughts), and uses `parseThought` to recover thought
   metadata from `text` parts marked `thought: true`.

The reconstruction also handles tool-call lookahead
(chatRecordingService.ts:919-945): when a `model` turn has
`functionCall` parts, it peeks at the next `user` turn for matching
`functionResponse.id` parts and groups sibling parts (text, inline
images) into the same tool result entry.

`geminiChat.ts:281-289` adds an analogous fix to `initialize`: if
history is provided but no `resumedSessionData`, sync the recording
service to the just-loaded history immediately.

`client.ts:327`, `client.ts:368`, and `environmentContext.ts:88`
relax `Content[]` to `readonly Content[]` to make the new sync paths
type-correct.

## What looks good

- Three regression tests pinning the actual bug (`should repopulate
  cachedConversation.messages when updating from history if cache is
  empty (regression)` at chatRecordingService.test.ts:1240-1276) plus
  the force-reconstruct path (lines 1278-1303) plus the sibling-parts
  case (lines 1305-1340). The "regression" suffix in the test name
  is the right call — it preserves the bug fingerprint forever.
- The sibling-parts handling at chatRecordingService.ts:925-941 is
  subtle but correct: providers can emit a tool-result turn that
  interleaves text, function responses, and inlineData (image
  outputs from a tool), and this code groups them by the *most
  recent* `functionResponse.id`. The matching test asserts all three
  parts (text, response, image) end up in `result[]`.
- `readonly Content[]` propagation at client.ts:327, 368 and
  environmentContext.ts:88 is the correct way to type-clean the new
  call paths without forcing copies.
- The `initialize` sync (geminiChat.ts:281-289) handles the
  resumedSessionData=undefined branch — the case where someone calls
  `startChat(history)` directly, not through resume — which is the
  same shape `/rewind` ends up in.

## Nits

1. `reconstructMessagesFromHistory` synthesizes `id: randomUUID()`
   and `timestamp: new Date().toISOString()` for every message
   (chatRecordingService.ts:867-871, 877-881). For *resumed* history
   this loses the original timestamps; subsequent log analysis will
   see all reconstructed messages clustered at resume time. If the
   `Content` doesn't carry an original timestamp (gemini-genai
   doesn't), at least mark the reconstruction with a flag (e.g.
   `reconstructedAt: ISO`) so downstream consumers can distinguish.
2. `CoreToolCallStatus.Success // Assume success for reconstructed
   history` (chatRecordingService.ts:891) is a load-bearing
   assumption. If the original tool call failed (the model still saw
   a `functionResponse` with an `error` field), this metric will
   be wrong. Consider inspecting the matched response part for an
   `error` key before defaulting.
3. The reconstruction loop ignores roles other than `user` and
   `model` (chatRecordingService.ts:953: `else { i++ }`). That's
   correct today, but if Gemini ever introduces a `system` or
   `tool` role in `Content`, those messages will silently disappear
   from the recording. A debug log would help future-debug.
4. `updateMessagesFromHistory(history, true)` overrides the empty-
   cache branch unconditionally (chatRecordingService.ts:909-916).
   That's fine for the documented call site (`setHistory` at
   geminiChat.ts:786), but the function now has three behaviors
   gated on two parameters — the next reader will appreciate a
   short doc comment.
5. The `slashCommandProcessor.ts` switch from `setHistory` to
   `resumeChat` (line 551) makes the call `await`ed inside a
   `case`. Verify the callers handle the new async-throw surface
   (`resumeChat` may now reject if `startChat` fails internally,
   where `setHistory` was synchronous).

## Risk

Medium. The reconstruction is the kind of "rebuild semantic state from
serialized form" code that tends to regress when the serialization
shape evolves. The current shape coverage in tests is good (tool
calls, sibling parts, thoughts), but a fuzz test that round-trips
arbitrary `Content[]` through `record → updateMessagesFromHistory →
read` would catch shape drift faster.

The user-facing impact (`/rewind` now actually rewinds the recording
view too) is a real bug fix and worth shipping. Land after addressing
the timestamp-synthesis nit, even if the others slip to a follow-up.
