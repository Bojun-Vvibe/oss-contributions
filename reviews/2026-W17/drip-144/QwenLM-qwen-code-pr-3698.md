---
pr_number: 3698
repo: QwenLM/qwen-code
head_sha: e789dc8d4f06ce84c49f7b88d375c3ccc1f69d0a
verdict: merge-after-nits
date: 2026-04-28
---

# QwenLM/qwen-code#3698 — fix(acp): run auto compression before model sends

**What changed.** +190/−40 across 2 files. Closes #3652.

`packages/cli/src/acp-integration/session/Session.ts` now wraps every direct `GeminiChat.sendMessageStream(...)` callsite with a pre-call to `geminiClient.tryCompressChat(sessionId, /*force*/false, abortSignal)` followed by a re-fetch of the chat instance via `geminiClient.getChat()` (because `tryCompressChat` can *replace* the chat instance — load-bearing detail, see test cell `uses the current chat after automatic compression replaces it` at `:325-352`). Test surface adds a `describe('auto-compress', ...)` block at `:300-444` with four cells:

1. **Positive baseline** at `:301-323`: prompt → assert `tryCompressChat` called with `(sessionId, false, AbortSignal)`, AND `tryCompressChat.invocationCallOrder[0] < sendMessageStream.invocationCallOrder[0]` — pinning the *order* invariant, not just both-were-called.
2. **Replace-chat cell** at `:325-352`: `tryCompressChat.mockImplementation(async () => { mockGeminiClient.getChat.mockReturnValue(compressedChat); ... })` then asserts `mockChat.sendMessageStream` was *not* called and `compressedChat.sendMessageStream` *was* — proves the re-fetch-after-compress contract.
3. **Tool-response follow-up cell** at `:354-410`: drives a tool call, then asserts `tryCompressChat` called *twice* (initial prompt + post-tool follow-up send), with the second call positionally `nthCalledWith(2, ...)` — closes the gap that the original CLI path also compresses before tool follow-ups, which is exactly where context blowup typically happens.
4. **Slash-command negative cell** at `:412-432`: `/compress` handled by `nonInteractiveCliCommands.handleSlashCommand` (no model send) → asserts `tryCompressChat` *not* called and `sendMessageStream` *not* called. Pins the discriminator that compression only runs when there's actually a model send pending.

**Why it matters.** PR root-cause writeup is correct: ACP's `Session` class invoked `GeminiChat.sendMessageStream()` directly, bypassing `GeminiClient.sendMessageStream()` where automatic compression runs on the regular CLI path. So ACP-driven sessions silently never compressed, accumulating until they hit the model's hard context limit and failed mid-turn. The fix promotes compression to a Session-level invariant rather than coupling it to the GeminiClient send path.

**Concerns.**
1. **Order-of-operations test at `:319-322`** is the highest-value cell here — `invocationCallOrder` is a vitest mechanism that pins ordering rather than just call presence. Without this cell, the diff could pass tests with compression running *after* the send (which would be a no-op on the next turn but break the current turn). Worth highlighting in code-review checklist.
2. **`tryCompressChat(sessionId, false, abortSignal)` second arg `false`** is the `force` parameter — meaning compression only runs when the heuristic says it should (token threshold), not on every message. That's the correct default; a `true` here would over-compress. Verify the threshold heuristic is the same on the ACP and CLI paths (if they diverge, ACP users get different compression behavior).
3. **Re-fetch via `geminiClient.getChat()` after each `tryCompressChat`** at the implementation site (in the diff continuation past the visible slice) is the load-bearing detail. The compressed-chat test cell verifies this works for the *first* send. Verify the tool-follow-up cell also re-fetches between the tool call and the follow-up send — if not, a compression triggered by the tool result would be dropped.
4. **`mockGeminiClient.tryCompressChat` mock returns `CompressionStatus.NOOP` by default** at the `:84-90` setUp, which is the right "compression considered, didn't fire" baseline. The replace-cell overrides to `CompressionStatus.COMPRESSED`. Worth a third state cell asserting `CompressionStatus.FAILED` doesn't crash the send path (compression failed, send proceeds with un-compressed history) — that's the operational failure mode worth pinning.
5. **`recordSlashCommand: vi.fn()` added to telemetry mock at `:117`** is required because the slash-command negative cell drives that path. Make sure the production telemetry recorder is also wired here, otherwise the slash-command branch silently drops telemetry.
6. **No assertion that `sessionId########1` thread-id format is stable** — `'test-session-id########1'` appears as a literal in three test cells. If the suffix-numbering scheme changes, three tests break. Consider extracting to a helper or asserting the prefix only.
7. **Test cleanup at `:166`** correctly nulls `mockGeminiClient` to prevent cross-test bleed.
8. **Closes a real bug class** — every multi-turn ACP user was silently losing context-management. The fix is structurally right (promote to Session invariant) and well-tested.

Right shape, real bug, excellent test coverage with order-pinning. Ship after adding the `CompressionStatus.FAILED` cell (concern 4) and confirming tool-follow-up re-fetches the chat (concern 3).
