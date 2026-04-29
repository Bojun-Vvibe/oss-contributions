---
pr: google-gemini/gemini-cli#26225
sha: ab4c6461dbba40c64dbac3607f70d5823432ecd1
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(core): fail fast in MessageBus.request() on publish failure

URL: https://github.com/google-gemini/gemini-cli/pull/26225
Files: `packages/core/src/agents/local-invocation.ts`,
`packages/core/src/confirmation-bus/message-bus.ts`,
`packages/core/src/confirmation-bus/message-bus.test.ts`,
`packages/core/src/scheduler/scheduler.ts`,
`packages/core/src/scheduler/state-manager.ts`,
`packages/core/src/tools/tools.ts`
Diff: 69+/47-

## Context

`MessageBus.publish()` previously swallowed validation/policy errors
into an `'error'` event and resolved successfully. That meant
`MessageBus.request()` (which awaits a response correlated to the
publish) hung forever when the publish itself failed validation, since
no response would ever arrive for a request that never made it to
subscribers. Every fire-and-forget caller had been working around this
by ignoring the publish promise; this PR flips the contract to "publish
throws on failure" and updates every callsite to either `.catch()` it
explicitly or propagate.

## What's good

- The contract flip at `message-bus.ts:147` adds `throw error;` after
  `this.emit('error', error)` — minimal, makes both signals available
  (event subscribers still see the error, awaiters now see the
  rejection). Order matters: emit-then-throw means error handlers run
  before the rejection propagates, preserving existing diagnostic
  logging.
- The fix at `message-bus.ts:243-247` in `request()` is the load-bearing
  piece — chains `.catch()` on the publish, which calls `cleanup()`
  (unsubscribes the response handler, clears the timeout) *and*
  rejects the outer request promise. Without that cleanup, a failed
  publish inside `request()` would leak the response subscription
  forever.
- Six callsite updates done symmetrically — every `void
  this.messageBus.publish({...})` becomes `void this.messageBus
  .publish({...}).catch(() => {})` (`local-invocation.ts:90-96`,
  `state-manager.ts:247-252`, `tools.ts:245-251`,
  `tools.ts:233-237`). The empty `.catch(() => {})` preserves the
  prior fire-and-forget semantics while satisfying the no-floating-
  promise lint.
- Scheduler's `handleToolConfirmationRequest` at `scheduler.ts:170-178`
  upgrades to a `try { await ... } catch` with `debugLogger.error(...)`
  — better than `.catch(() => {})` because confirmation-response
  publish failures are diagnostically interesting (they indicate a
  malformed confirmation response, not just a transient bus issue).
- Test updates at `message-bus.test.ts:34-44,49-61,255-258` flip the
  assertions from `.resolves.not.toThrow()` to `.rejects.toThrow(...)`
  with the exact error message strings — locks the new contract,
  catches a future "swallow errors silently again" regression, and
  the `'Invalid message structure'` / `'Policy check failed'`
  assertions are specific enough to catch error-message refactors that
  would break downstream error-classification logic.

## Nits

- The empty `.catch(() => {})` pattern at four call sites is correct
  but loses the diagnostic information that was in the swallowed
  error. Consider promoting at least the
  `subagent_activity` / `tool_calls_update` / `update_policy` paths
  to `.catch((err) => debugLogger.warn('Failed to publish X:', err))`
  — these are fire-and-forget but a sustained publish failure is a
  bus-health signal worth observing in debug logs.
- `tools.ts:233-237` switched from `try { void publish(request); }
  catch { cleanup(); resolve('allow'); }` to `publish(request).catch(
  () => { cleanup(); resolve('allow'); })`. The semantics changed
  subtly: the old code only caught synchronous throws (e.g., schema
  rejection thrown immediately by the policy engine); the new code
  catches both sync and async failures including transient bus errors
  → and resolves to `'allow'` (the permissive fallback). Fine if
  fail-open is the policy here, but worth a comment naming the
  decision since fail-open on confirmation requests has security
  implications.
- The scheduler upgrade to `try/await/catch` while other call sites
  use `.catch(() => {})` is inconsistent. Either standardize on the
  awaited form (cleaner control flow, easier to add metrics) or
  explicitly comment why fire-and-forget is correct for those four
  paths and not for `handleToolConfirmationRequest`.
- The `request()` cleanup-then-reject ordering at `:243-247` is
  correct; consider an additional test exercising "publish throws,
  awaiter sees rejection, response subscription is gone" — the
  current test suite covers the event-emission side but not the
  subscription-leak side which is the actual bug class this PR
  fixes.
- Removed `eslint-disable-next-line @typescript-eslint/no-floating-promises`
  is good cleanup; verify no other call sites still have it for the
  same reason.

## Verdict reasoning

Correctly-shaped contract flip from "swallow + emit" to "emit +
throw" with the load-bearing `request()` cleanup wired correctly
(no subscription leak when publish fails inside request). All call
sites updated symmetrically. Test assertions upgraded to lock the
new contract. Nits center on observability (empty `.catch()` loses
diagnostic info), the fail-open semantics in `tools.ts:233`, and
inconsistent error-handling shape between the five updated sites.
None blocking; merge after addressing the fail-open comment and
optionally upgrading two-three of the empty catches to debugLogger.
