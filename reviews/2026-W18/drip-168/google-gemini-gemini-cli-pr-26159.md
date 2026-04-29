# google-gemini/gemini-cli PR #26159 — feat(core): implement continuation auto-recovery

- Repo: google-gemini/gemini-cli
- PR: https://github.com/google-gemini/gemini-cli/pull/26159
- Head SHA: `eb1527785a588da481e691b20e546d62d91003f9`
- Author: achernez
- Size: +145 / −3, 3 files

## Context

When the model returns `finishReason: MAX_TOKENS` mid-response, the CLI
previously emitted whatever it got and stopped, leaving the user with a
truncated answer and no recovery path other than typing "continue" by
hand. This PR adds an auto-continuation loop in `LegacyAgentProtocol` that
detects `MAX_TOKENS` and re-issues a synthetic user turn
(`[System: Your previous response was truncated due to length. Please
continue exactly where you left off.]`), capped at 3 continuations per
original turn.

## What the diff actually does

Production change at `packages/core/src/agent/legacy-agent-session.ts:177-275`:

1. New loop-local state at `:179-180`:
   `let continuationCount = 0; const maxContinuations = 3;`
2. Inside the response stream consumer at `:227-235`, the
   `GeminiEventType.Finished` case now captures `finishReason =
   event.value.reason` and only `_finishStream(...)`s when both
   `toolCallRequests.length === 0` *and* `finishReason !==
   FinishReason.MAX_TOKENS`. Previously it bailed on any
   `Finished + no tool calls`.
3. After the stream consumer at `:251-274`, the post-loop `if
   (toolCallRequests.length === 0)` branch now checks for
   `finishReason === FinishReason.MAX_TOKENS && continuationCount <
   maxContinuations`. If so: increments `continuationCount`, emits a
   user-message event with `_meta: { isContinuation: true }`,
   replaces `currentParts` with `[{ text: continuationPrompt }]`, and
   `continue`s the outer `while (true)`.
4. On the third continuation (or any non-MAX_TOKENS finish with no tool
   calls), `_finishStream(mapFinishReason(finishReason) || 'completed')`
   resolves correctly.

Type changes at `packages/core/src/agent/types.ts:79-83, 319-321`:
- `AgentEventCommon._meta` gains optional `isContinuation?: boolean`.
- `Usage` gains optional `isContinuation?: boolean` (forward-compat for
  surfacing per-turn continuation counts in cost reporting).

Tests at `packages/core/src/agent/legacy-agent-session.test.ts:1417-1525`:
- "triggers continuation when finishReason is MAX_TOKENS" — mocks
  `sendMessageStream` twice (first returns `MAX_TOKENS + 'part 1'`,
  second returns `STOP + 'part 2'`), asserts 4 messages emitted (user
  start, agent part 1, user continuation prompt with `_meta:
  {isContinuation: true}`, agent part 2), and pins that the second
  `sendMessageStream` call carries the continuation prompt as a part.
- "stops after maxContinuations (3)" — mocks 5 `MAX_TOKENS` returns in
  a row, asserts `sendMock` was called exactly 4 times (1 original + 3
  continuations).

## Risks and gaps

1. **The continuation prompt is a string literal, not a localized or
   injected template**. `'[System: Your previous response was
   truncated due to length. Please continue exactly where you left
   off.]'` is reasonable English but (a) won't be in the message-bundle
   pipeline if the project has one, and (b) some non-English models
   respond more reliably to a continuation hint in the conversation
   language. A `Config`-injectable continuation prompt (with the literal
   as default) would let multilingual deployments tune it.

2. **`continuationCount` resets per outer turn but never resets across
   the session lifetime**. That's correct in this PR's scope — the
   counter is loop-local — but it means a user who hits MAX_TOKENS on
   turn N then again on turn N+1 gets a fresh 3 attempts each time. If
   that's intentional, fine. If the goal is "no more than 3
   auto-continuations per session as a runaway-cost guard", the counter
   needs to live on `LegacyAgentSession` not on the `processCommand`
   stack.

3. **The continuation user-message is emitted *into the conversation
   history* via `this._emit([continuationMessage])`**, which means the
   model sees the synthetic prompt on the next request and downstream
   transcript renderers will display it as a literal user turn. Two
   followups: (a) does the transcript UI know to render
   `_meta.isContinuation` differently (collapsed, italic, hidden), and
   (b) does session export / fork carry the synthetic message into the
   forked context (probably yes — and that's mostly fine, but if anyone
   replays a forked session against a different model they'll see the
   weird literal `[System: ...]` user turn).

4. **The continuation prompt is wrapped in `[System: ...]` brackets but
   sent as a regular `user` turn**, not as a system message. Most
   model APIs distinguish system from user, and the visible bracket
   convention is doing the work of pretending. For Gemini this is
   probably fine because a true mid-conversation system message isn't
   well-supported by the API anyway. For non-Gemini models routed
   through the same path (do they exist in this codebase?), the
   distinction may matter.

5. **`maxContinuations = 3` is hardcoded**, not a `Config` knob. For
   power users on models with very small per-call output limits, 3 may
   not be enough; for batch scripts with cost ceilings, 3 may be too
   many. A `Config.getMaxContinuations()` with default `3` would close
   this without breaking anyone.

6. **The test for "stops after maxContinuations (3)"** asserts
   `sendMock.toHaveBeenCalledTimes(4)` — that's correct (1 original +
   3 continuations), but it doesn't assert the user sees a final
   "response was truncated and continuation budget exhausted" hint.
   Currently the user just gets `_finishStream(mapFinishReason(MAX_TOKENS))`
   with no signal that the system tried to recover and gave up. A
   one-line UI hint (or at minimum a debug-logger entry) on the
   give-up path would help debugging.

7. **`isContinuation?: boolean` in `Usage`** is added but the diff
   visible doesn't show it being populated anywhere. If it's intended
   as a forward-compat hook, fine; if it's expected to be set by this
   PR, the wiring is missing. Worth confirming or removing for now to
   avoid a typed-but-never-set field that misleads downstream
   consumers.

## Suggestions

- Make `maxContinuations` a `Config` knob with default 3.
- Consider externalizing the continuation prompt string for
  localization / model-tuning, with the literal as default.
- Add a UI hint or `debugLogger.warn` on the give-up path so
  exhausted-budget cases are observable.
- Decide whether `Usage.isContinuation` should be set in this PR or
  removed for now to avoid a stale typed-but-unwired field.
- Confirm transcript renderers handle `_meta.isContinuation` (collapse,
  italic, hide) and add a snapshot test.
- Consider whether the continuation prompt should be sent as `system`
  rather than `user` for non-Gemini providers routed through the same
  path.

## Verdict

**merge-after-nits** — the loop logic is correct (per-turn-local counter,
proper guard against tool-call interleaving by checking
`toolCallRequests.length === 0` before the continuation branch), and the
two tests pin both the happy path and the cap. Nits are about
configurability, observability of the give-up case, and the unused
`Usage.isContinuation` field, none of which block merge.
