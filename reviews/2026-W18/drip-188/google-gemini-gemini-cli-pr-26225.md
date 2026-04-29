# google-gemini/gemini-cli#26225 — fix(core): fail fast in MessageBus.request() on publish failure

- PR: https://github.com/google-gemini/gemini-cli/pull/26225
- HEAD: `ab4c646`
- Author: Adib234
- Files changed: 6 (+69 / -47)

## Summary

Pre-fix, `MessageBus.publish` swallowed validation/policy errors via
`emit('error', error)` without re-throwing, and `MessageBus.request` did
a fire-and-forget `void publish(...)`. Net effect: a request that failed
its policy check or schema validation would set up a response listener
that *never resolved*, producing a hang until the surrounding tool
timeout (often 30s+). The fix: `publish` now `throw`s after `emit('error',
...)`; `request` awaits the publish promise and on rejection calls
`cleanup()` and `reject(error)`. All other `void publish(...)`
fire-and-forget call sites get explicit `.catch(() => {})` to preserve
their intentional ignore-on-error semantics now that the underlying
promise actually rejects.

## Cited hunks

- `packages/core/src/confirmation-bus/message-bus.ts:144-148` — the
  catch block now does `this.emit('error', error); throw error;`
  instead of swallowing. This is the load-bearing change; without it
  the emit-only path swallows the rejection and downstream awaiters
  never know.
- `packages/core/src/confirmation-bus/message-bus.ts:240-247` — `request`
  no longer fires `this.publish(...)` and discards. New code:
  `this.publish(...).catch((error) => { cleanup(); reject(error); })`.
  `cleanup` correctly removes the response subscription before rejecting
  so the next `request` call doesn't race against a dead handler.
- `packages/core/src/agents/local-invocation.ts:91-99` — `publishActivity`
  was a true fire-and-forget; now adds `.catch(() => {})` so an invalid
  activity message doesn't surface as an unhandled rejection. This is
  the right call — subagent activity publishing is best-effort
  telemetry-style, not control flow.
- `packages/core/src/scheduler/state-manager.ts:246-251` — same
  `.catch(() => {})` on `TOOL_CALLS_UPDATE` publish. Comment correctly
  notes "Fire and forget - The message bus handles the publish and
  error handling" — `.catch` preserves that intent now that publish
  rejects.
- `packages/core/src/scheduler/scheduler.ts:170-179` — the
  `handleToolConfirmationRequest` path is *not* fire-and-forget — it
  now wraps the `await publish(...)` in `try/catch` and on failure logs
  to `debugLogger.error('Failed to publish confirmation response',
  error)`. Important: this is a response-publish in reaction to an
  inbound request, so silently dropping the publish would leave the
  original requester awaiting forever. The debug log isn't ideal
  (operator may not see it), but at least the failure is observable.
- `packages/core/src/tools/tools.ts:245-254, 366-370` — two more
  fire-and-forget conversions plus one *non*-fire-and-forget conversion.
  At `:366-370` the old code had `try { void publish(request); } catch
  { cleanup(); resolve('allow'); }` — a synchronous-only catch that
  couldn't see async rejections. New code:
  `this.messageBus.publish(request).catch(() => { cleanup();
  resolve('allow'); })` — now actually catches the async rejection and
  fails open to `'allow'`. Whether *that* default is correct is a
  policy question (failing closed to `'deny'` would be safer for a
  confirmation request that couldn't be published), but the change
  faithfully preserves the prior intent.
- `packages/core/src/confirmation-bus/message-bus.test.ts:33-41,
  46-58, 254-260, 290-301, 337-348` — three test updates. Two
  `// @ts-expect-error` invalid-message tests now use
  `as unknown as Message` casts and assert
  `.rejects.toThrow('Invalid message structure')` — locks the new
  contract that publish *throws* on validation. The policy-rejection
  test at `:254-260` flipped from "should not throw" to "should throw
  with 'Policy check failed'" — same intent, asserts that the new
  throw-after-emit pattern surfaces denied requests as rejections.
  Two test-internal handlers got `.catch(() => {})` to match the new
  fire-and-forget convention.

## Risks

- **Failing open to `'allow'`** in `tools.ts:366-370` is a behaviour
  change with security implications. Old code: synchronous throw → fall
  through to `resolve('allow')`. New code: async rejection → fall
  through to `resolve('allow')`. The new path *also* triggers when
  policy denies the request (since publish now rejects on policy
  failure), which means a policy-denied confirmation now resolves to
  `'allow'` where previously it might have hung. That's plausibly
  *less* safe than the hang. Recommend changing this fallback to
  `'deny'` (or making the fallback configurable) before merging.
- The `debugLogger.error` at `scheduler.ts:174` is fired for failures
  that previously hung silently. Operators currently filtering for the
  hang symptom (request timeout) will now see error log lines instead;
  log-volume dashboards should be sanity-checked.
- `MessageBus extends EventEmitter`, and EventEmitter's `'error'` event
  has special semantics (throws synchronously if no listener). Now that
  `publish` *also* throws, an `'error'` listener that itself throws
  would surface twice (once via the listener crash, once via the
  publish throw). Worth a unit test asserting the double-fault path
  doesn't crash the process.
- The `as unknown as Message` casts in tests trade compile-time safety
  for runtime test coverage. That's the right call here (the tests
  *want* to publish invalid shapes), but a comment noting "we can't
  use ts-expect-error because the rejected-publish assertion is async"
  would prevent future cleanup from regressing back.
- The test at `:254-260` switched from `resolves.not.toThrow()` to
  `rejects.toThrow('Policy check failed')` — consumers depending on
  the prior "publish always resolves" contract are now broken. This is
  a public behaviour change of `MessageBus.publish` and deserves a
  CHANGELOG / migration-note entry.

## Verdict

**merge-after-nits**

## Recommendation

Flip the fail-open `resolve('allow')` at `tools.ts:368` to `'deny'`
before landing — failing closed on a publish error preserves the safer
default. Add a CHANGELOG entry calling out the publish-now-throws
contract change; the underlying hang fix is correct and exactly the
right shape (request-response surface should never silently lose its
producer).
