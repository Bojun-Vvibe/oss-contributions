# google-gemini/gemini-cli #26498 — feat(cli): show acknowledgment when user steering hint is processed

- **Head SHA:** `9e83545ac3645da5b5627d9bfc6940a1acf70326`
- **Base:** `main`
- **Author:** euxaristia
- **Size:** +142 / −6 across 4 files
  (`packages/cli/src/test-utils/fixtures/steering.responses`,
  `packages/cli/src/ui/hooks/useGeminiStream.ts`,
  `packages/core/src/utils/fastAckHelper.test.ts`,
  `packages/core/src/utils/fastAckHelper.ts`)
- **Verdict:** `merge-after-nits`

## Summary

When the user types a steering hint mid-task, the CLI was silently
prepending it to the model's next prompt with no UI feedback. This PR
makes a parallel `generateSteeringAckMessage()` call that produces a
short "Understood. Adjusting the plan." style acknowledgment line and
renders it as an INFO history item. Also tightens prompt-injection
sanitization (escapes `</tag>` and `]` in wrapped user/background
content) and plumbs an external `AbortSignal` through
`generateSteeringAckMessage` so a user-cancelled stream cancels the
ack request too.

## What's right

- **Ack is fire-and-forget, doesn't block the stream.** At
  `useGeminiStream.ts:2005-2025` the ack is launched with `void
  generateSteeringAckMessage(...).then(...)` — the next-turn prompt
  with the wrapped hint still gets pushed to `responsesToSend` (line
  2002) before the async ack starts. So the model continues working
  immediately; the ack is purely UI feedback. Critical: a slow ack
  call won't gate the actual response.

- **Ack uses the parent stream's abort signal.** The signal pulled
  from `abortControllerRef.current?.signal` (line 2007) is forwarded
  via the new `options.signal` parameter on
  `generateSteeringAckMessage` (`fastAckHelper.ts:125`). When the user
  cancels the turn, both the main stream and the ack request abort.
  Without this, an aborted turn could still surface a stale "Understood"
  ack a few seconds later — exactly the kind of UI confusion the
  feature is trying to *avoid*.

- **Listener cleanup is correct.** `fastAckHelper.ts:139-141, 154`
  uses `addEventListener('abort', ..., { once: true })` and a paired
  `removeEventListener` in `finally`. The unit test at
  `fastAckHelper.test.ts:218-238` (`removes the abort listener after
  generation completes`) explicitly asserts both that the listener is
  added with `{ once: true }` and that the *same handler reference*
  is removed. That's the correct way to verify no listener leak under
  repeated steering hints.

- **Pre-aborted signal short-circuits before any LLM call.**
  `fastAckHelper.ts:131-133` returns the fallback text immediately if
  `options.signal?.aborted` is true at entry. Test at
  `fastAckHelper.test.ts:152-167` confirms `llmClient.generateContent`
  is never called in that case. Cheap optimization that also avoids
  burning a call against rate limits when the user abort-spammed.

- **Prompt-injection sanitization is centralized.** The new
  `sanitizeForWrapper()` helper (`fastAckHelper.ts:62-67`) is applied
  by both `wrapInput()` and `wrapBackgroundOutput()` (lines 71, 75).
  It escapes `</tag>` (XML closing tag) and `]` (context breaker for
  bracket-delimited LLM templates). Tests at
  `fastAckHelper.test.ts:171-209` cover four cases: closing-tag
  escape in steering hint, in user-hint list, in background output,
  and multi-tag inputs. The `]` escape (lines 199-209) catches a
  failure mode that was previously unguarded — a malicious tool output
  containing `]` could break out of a `[step 1] [step 2]` formatted
  context.

- **Test fixture update is honest.** The new line in
  `steering.responses` (line 2) adds the
  `generateContent`-shaped response that the new ack call expects, so
  the integration test for steering still passes deterministically.
  Forgetting this fixture row would have surfaced as a flaky test.

## Nits / before-merge

1. **Catch handler swallows non-abort errors silently.**
   `useGeminiStream.ts:2022-2025` does `.catch((err) => { if
   (err?.name === 'AbortError') return; /* silently ignore */ })`. For
   a non-critical UI feature this is the right default, but a debug
   log (`logger.debug('steering ack failed', err)`) would help track
   silent regressions if the ack stops appearing for some users. The
   inline comment ("Silently ignore — steering ack is non-critical UI
   feedback") is honest about the choice, so this is a soft nit.

2. **`addItem` runs from a `.then()` callback after the turn may have
   moved on.** The ack uses `ackTimestamp = Date.now()` captured
   *before* the LLM call, which preserves chronological order in the
   history. But if the model has already streamed a long response by
   the time the ack resolves, the ack might appear visually
   *interleaved* with model output rather than at the moment the user
   typed. The signal-cancel mitigates the case where the user cancels,
   but not the case where the model just responds quickly. Worth
   testing: if the ack callback's `addItem` lands while
   `setIsResponding` is still true, does the renderer order it
   correctly? The `marginBottom: 1` styling suggests this was
   considered.

3. **`config` added to dep array is correct but unexplained.** Line
   2067 adds `config` to the `useCallback` deps. Necessary because
   `config.getBaseLlmClient()` is now read inside the callback, but a
   one-line comment would help future readers who come across the new
   dep.

4. **Test import path change `import { LlmRole } from
   'src/telemetry/llmRole.js'` →
   `'../telemetry/llmRole.js'`** (`fastAckHelper.test.ts:14`) is a
   drive-by but a *correct* one. The `src/...` form was working only
   by accident of TS path mapping; relative is the project's
   convention. Approve, but note that this kind of import-path
   normalization is the sort of thing that ideally lands in its own
   PR for clean revertability.

5. **`STEERING_ACK_INSTRUCTION` not visible in the diff but
   referenced.** The change adds an `options?.signal?.addEventListener`
   path inside `generateSteeringAckMessage` but doesn't show
   `STEERING_ACK_INSTRUCTION` itself. Reviewers should spot-check that
   the instruction text constrains the model output to single-sentence
   acknowledgments — otherwise a chatty model could produce a
   multi-line message that disrupts the streaming UI flow.

## Risk

- **Latency budget:** the ack call uses a separate `STEERING_ACK_TIMEOUT_MS`
  (existing) and now also responds to the parent abort. So the
  upper-bound additional latency is `min(timeout, parent_signal_age)`
  worth of one extra LLM round-trip — negligible vs the main stream,
  and entirely off the critical path.
- **Cost:** every steering hint now triggers a second LLM call. For
  high-frequency steering users this is a measurable cost increase.
  Worth verifying with the team that this is intentional (and that
  the fast-model routing applies — see also #3848 in qwen-code for a
  similar fast-model routing pattern).

## Verdict

`merge-after-nits` — the abort plumbing and sanitization fixes are
both well-tested and address real failure modes. Nits are mostly
documentation/observability; logging the swallowed error is the only
one I'd push for before merge.
