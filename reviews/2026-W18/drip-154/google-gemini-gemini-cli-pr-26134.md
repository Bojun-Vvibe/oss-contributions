# google-gemini/gemini-cli #26134 — fix(cli): clear pending steering hint buffer in cancelOngoingRequest

- PR: https://github.com/google-gemini/gemini-cli/pull/26134
- Head SHA: `1a86e944d546e83619afc4a697f0b858482df6f5`
- Files: `packages/cli/src/ui/AppContainer.tsx` (+1), `packages/cli/src/ui/hooks/useAgentStream.ts` (+3 / -1), `packages/cli/src/ui/hooks/useAgentStream.test.tsx` (+19), `packages/cli/src/ui/hooks/useGeminiStream.ts` (+3 / -0), `packages/cli/src/ui/hooks/useGeminiStream.test.tsx` (+43)
- Fixes: #26133

## Citations

- `AppContainer.tsx:1206-1208` — at the `useAgentStream` callsite (the non-Gemini codepath used for ACP-protocol agents), passes `consumeUserHint: consumePendingHints` into the hook options. The same `consumePendingHints` was already wired into the `useGeminiStream` callsite via the existing options destructuring; this PR completes the symmetry.
- `useAgentStream.ts:43-46` — `UseAgentStreamOptions` adds `consumeUserHint?: () => string | null`. Optional with the right contract: returns the hint (and *consumes* it from the pending buffer as a side effect) or null.
- `useAgentStream.ts:127-131` — inside `cancelOngoingRequest`, after `await agent.abort()` and `setStreamingState(StreamingState.Idle)` and `onCancelSubmit(false)`, the new line `consumeUserHint?.();` discards the return value. The intent is to *drain* the pending-hint buffer; the value isn't used because the user-pressed-Esc-during-streaming path is "throw away whatever was queued." Optional chain `?.()` is correct because not every test/host wires this prop.
- `useAgentStream.ts:131` — `consumeUserHint` added to the `useCallback` deps array. Without this, React would close over the first-render reference of `consumeUserHint` and a hot-reloaded host that swapped the function out wouldn't see the new one. Right hygiene.
- `useGeminiStream.ts:881-883` — same pattern at the symmetric Esc-handler location: after `onCancelSubmit(false)` and `setShellInputFocused(false)`, calls `consumeUserHint?.()`. Added to the `useCallback` deps array at line 893.
- `useAgentStream.test.tsx:208-225` — new `it('should clear pending user hints when cancelOngoingRequest is called', ...)`. Spy: `vi.fn().mockReturnValue('some hint')` (the spy returns a plausible value so a future "hint must be empty to count as consumed" check would catch it). Asserts `consumeUserHintSpy.toHaveBeenCalled()` after `act(async () => { await result.current.cancelOngoingRequest(); })`.
- `useGeminiStream.test.tsx:1684-1722` — new `it('should clear pending user hints when escape is pressed', ...)` covers the user-facing path. Uses a never-resolving stream (`yield { type: 'content', value: 'Part 1' }; await new Promise(() => {})`) to force `streamingState = Responding`, calls `submitQuery`, then `simulateEscapeKeyPress()`, asserts `consumeUserHintSpy.toHaveBeenCalled()`. Importantly it doesn't assert the queue *value* was discarded — relies on `consumeUserHint`'s contract that calling it consumes the buffer.

## Verdict

`merge-as-is`

## Reasoning

This is a one-symptom-two-line bug fix with the right test discipline. The bug (#26133) is the kind of UX paper cut that's hard to reproduce on demand: user types a steering hint into the input while the agent is mid-response, presses Esc to cancel the in-flight turn, and on next prompt the hint they thought they discarded gets auto-prepended to their next message. The internal model is "Esc = throw everything away," but `cancelOngoingRequest` was only aborting the network turn and resetting streaming state — the pending-hint buffer (a separate piece of state owned by `consumePendingHints`) survived the cancellation and was consumed on the *next* user submission instead.

The fix is symmetric across both stream hooks: `useGeminiStream` (the standard Gemini code path) and `useAgentStream` (the ACP-protocol path used for non-Gemini agents like Claude/GPT through the agent proxy). That symmetry is critical — without it, users who'd switched their default to a non-Gemini agent would still hit the bug because only the Gemini path got the fix. The shared call-site at `AppContainer.tsx:1206-1208` (where the conditional picks one hook or the other based on agent type) is the right point to thread the same `consumePendingHints` callback through both, and the optional `?.()` calling convention means tests that don't wire the prop don't blow up.

The test pattern is the right shape:
1. **Spy-and-check-called, not value assertion.** `consumeUserHintSpy.toHaveBeenCalled()` is correct because the contract being tested is "the cancellation handler calls the consumer," not "the consumer returns nothing." Asserting the return value would couple the test to `consumePendingHints`'s implementation (which is owned by a different module).
2. **The Gemini-side test exercises the full keypress path.** `simulateEscapeKeyPress()` goes through `useKeypress` → handler → `cancelOngoingRequest` → consume. This pins the wiring all the way out to the keyboard event, not just the function call. Future refactors that move the consume call to a different cancellation surface (e.g. Ctrl-C handler) would catch the regression here.
3. **Both tests use a never-resolving stream / mock setup that forces the hook into the streaming-active branch.** This is the only state where `cancelOngoingRequest` actually runs the new code — testing it in idle state would silently pass because of the guard at the top of the existing handler.

Nits I'd note for the maintainer (none blocking):

- The PR doesn't add a test asserting `consumeUserHint` is called *exactly once* per cancel — if a future bug re-introduces a double-call (say, both the Esc handler and a separate StreamingState-effect both consuming), the spy would still pass `toHaveBeenCalled()`. Cheap to tighten with `toHaveBeenCalledTimes(1)`.
- `consumeUserHint` is typed `() => string | null` but the return value is discarded. If the type is genuinely `() => void` from the caller's perspective, consider widening the prop type or adding a comment explaining why the value isn't used (so a future reader doesn't think they should plumb it somewhere).
- The fix doesn't address Ctrl+C — only the Esc path was wired through `cancelOngoingRequest`. If the host has a separate SIGINT handler that aborts the stream without calling this function, the bug persists for terminal-power users. Worth a quick verification that Ctrl+C funnels through the same handler; if not, file as a sibling PR.

Net: tight fix, real bug, symmetric across both hooks, the right tests. Ship.
