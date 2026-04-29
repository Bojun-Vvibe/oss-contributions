# google-gemini/gemini-cli#26194 — test(core): add regression test for ToolConfirmationResponse payload

- PR: https://github.com/google-gemini/gemini-cli/pull/26194
- Head SHA: `ec0ca555fec7a0b3b7ff7ddcb92399c53a97f6a3`
- Author: Adib234
- Base: `main`
- Size: +39 / -0 in 1 file

## Summary

Pure regression test for issue #22120 — a previously-fixed bug where
`CoreToolScheduler.handleConfirmationResponse` dropped the `payload`
parameter. The bug was fixed during the migration to the event-driven
scheduler at `packages/core/src/scheduler/confirmation.ts`; this PR adds
a unit test in `confirmation.test.ts:360-396` to lock that behavior in.

## Notable design choices

- `confirmation.test.ts:362-367` — uses an `ask_user` confirmation
  details type with `questions: []` and `title: 'Title'` plus a mocked
  `onConfirm: vi.fn()`. Minimal viable shape.
- `confirmation.test.ts:374-381` — calls `resolveConfirmation()` with
  the full context object (config, messageBus, state, modifier,
  preferred-editor getter, scheduler ID), then `await
  listenerPromise` to ensure the bus listener is wired before
  emitting the response. Correct ordering — emitting before the
  listener attaches would silently drop the event.
- `confirmation.test.ts:385-391` — the `payload = { answers: { '0':
  'user choice' } }` shape mirrors a realistic `ask_user` response
  (indexed by question position), and the `correlationId`
  `'123e4567-e89b-12d3-a456-426614174000'` is a canonical UUID — fine
  for a regression test, just don't depend on the exact value.
- `confirmation.test.ts:393-396` — final assertion is precise:
  `expect(details.onConfirm).toHaveBeenCalledWith(
   ToolConfirmationOutcome.ProceedOnce, payload)`. Both arguments
  matter — outcome and payload — and the test will fail loudly if
  either is dropped.

## Concerns

1. **Single test for a multi-shape API.** `ToolConfirmationResponse`
   carries payloads for `ask_user`, `edit`, `exec`, and other
   confirmation types. The original bug (per the issue) was at the
   *response handling* layer, so it likely affected all payload
   shapes uniformly — but a strict regression test would parameterize
   over each confirmation `type` and prove the payload survives for
   each. As-is, a future refactor that adds an `if (type === 'edit')
   { /* drops payload */ }` branch would not be caught.
2. **Doesn't assert payload identity vs equality.** If the scheduler
   ever clones/reserializes the payload through JSON, `toHaveBeenCalledWith`
   with `expect.objectContaining(...)` semantics would still pass even
   though the call site receives a different object reference. For
   regression purposes this is fine — the bug was *omission*, not
   *mutation* — but worth noting.
3. **`correlationId` hardcoded in test, not derived.** If the
   scheduler ever validates that the correlation ID on the response
   matches the one it generated for the request, this test would
   regress only if the scheduler also added correlation-ID validation
   (which would be a separate change). Today the test just emits a
   canonical UUID and the scheduler accepts it — fine, but one more
   case where parameterization would help.
4. **Test placement.** Sits at line 360 inside the existing
   `describe('confirmation.ts', ...)` block, after another existing
   test. Naming convention `'should pass payload to onConfirm
   callback (Regression for #22120)'` matches `'should resolve
   immediately if IDE confirmation resolves first'` style — good.
5. **Pre-merge checklist `Validated on required platforms/methods`
   only ticks MacOS + npm run.** For a pure unit-test PR this is
   acceptable — the test runs in vitest under any Node version — so
   the unchecked Windows/Linux boxes aren't blockers. Reviewer can
   note this in the merge.

## Verdict

**merge-as-is** — small, focused, correct regression test for a
real bug. The single-confirmation-type coverage (item 1) is the only
substantive gap, and even that's a reasonable scope for a
"regression test for issue X" PR rather than "comprehensive
confirmation-response coverage." The core assertion is precise and
will fail loudly if the bug returns. No code under test is
modified; risk is zero. Ship it.
