# google-gemini/gemini-cli #26225 — fix(core): fail fast in MessageBus.request() on publish failure

- **Head SHA:** `2b9ce09b6e406959a96bdb150c9969392290f7ef`
- **Base:** `main`
- **Author:** Adib234
- **Size:** +246 / −40 across 7 files (1 test rig, 5 core files, 1 large test file)
- **Verdict:** `merge-after-nits`

## Summary

Fixes #22588: `MessageBus.request()` would silently hang for the full timeout
(60s by default) when `publish()` rejected, because the `request()` body
awaited the response listener without attaching a `.catch()` to the publish
promise. PR rewires four things together: (a) `request()` now fails fast on
publish rejection, (b) `publish()` re-throws after emitting on `'error'` so
async callers can observe failure, (c) sensitive args are sanitized via
`sanitizeToolArgs` before they hit debug logs or error messages, (d) every
floating `void messageBus.publish(…)` call site is rewritten to a
`const p = …; if (p instanceof Promise) p.catch(() => {});` pattern that
tolerates test mocks returning `undefined` or throwing synchronously.

## What's right

- **The actual bug fix is precise and well-tested.** The new test at
  `packages/core/src/confirmation-bus/message-bus.test.ts:328-353` mocks
  `publish` to reject, calls `request(…, 60000)`, and asserts both that the
  promise rejects with `'Publish failed'` AND that
  `Date.now() - start < 1000` — pinning that the request neither hangs nor
  consumes the 60s timeout budget. This is the correct shape of regression
  test for the reported bug.

- **Re-throw in `publish()` is the right contract change.** Previously
  `publish()` swallowed errors after emitting `'error'`, so awaiting callers
  couldn't distinguish "no error" from "error happened, you should look at
  the bus". The 22 existing test updates in `message-bus.test.ts:32-43`
  (`await expect(messageBus.publish(invalid)).rejects.toThrow('Invalid
  message structure')`) reflect the new contract uniformly. The
  `policy.test.ts` update at line 314 is the only behavioral test that
  flipped from `.resolves.not.toThrow()` to `.rejects.toThrow('Policy check
  failed')` — that's the right direction for surfacing policy denials to
  callers.

- **Sanitization at the right boundary.**
  `message-bus.test.ts:171-227` has two new tests pinning that secrets
  (`password: 'secret-password'`, `api_key: 'my-api-key'`) get `[REDACTED]`
  before either (a) the rejected error message reaches the caller via
  `rejects.toThrow(/\[REDACTED\]/)` AND `rejects.not.toThrow(/secret-password/)`,
  or (b) the debug logger receives the payload via `debugSpy` assertions.
  Both directions matter: a stack trace that bubbles up to a host's logging
  pipeline and a debug log that ends up in a captured trace are equally
  bad places to leak secrets.

- **The `if (p instanceof Promise) p.catch(() => {})` pattern is
  defensive in the right way.** It handles three real cases:
  (a) production where `publish` returns a real promise — `.catch()` keeps
  the unhandled-rejection from crashing the Node process (this is the
  hardening called out in the PR description),
  (b) test mocks that return `undefined` (the `vi.spyOn(messageBus,
  'publish').mockReturnValue(undefined)` pattern that several existing
  tests use),
  (c) test mocks that throw synchronously (`mockImplementation(() => {
  throw … })`).
  The two new tests at `message-bus.test.ts:421-466` ("pattern resiliency")
  pin both (b) and (c) directly so a future refactor that changes the
  pattern will visibly fail.

- **Call-site rewrites are consistent.** Five floating-publish sites get
  the same treatment: `AppRig.tsx:611-619`, `local-invocation.ts:92-103`,
  and three sites in `message-bus.test.ts:447-462`/`:489-501`. The
  `local-invocation.ts` site additionally wraps in `try { … } catch { /*
  Ignore errors in fire-and-forget activity update */ }` because subagent
  activity is best-effort by design — that's a reasonable distinction from
  the test-rig and message-bus internal sites where errors are swallowed
  with a comment-less `.catch(() => {})`.

## Concerns / nits

1. **The `try { … } catch { … } / if (p instanceof Promise) p.catch()`
   double-handler at `local-invocation.ts:93-107` is slightly redundant.**
   `messageBus.publish` either (a) throws synchronously (caught by the outer
   `try`), (b) returns a Promise that rejects (caught by `p.catch(() =>
   {})`), or (c) returns `undefined` (no-op). The two handlers don't
   overlap so this is correct, but a one-line comment explaining "outer
   try guards sync-throw from validation; inner catch guards async-reject
   from transport" would head off a future "this is redundant" cleanup PR.

2. **The 5 call-site rewrites duplicate the same boilerplate.** Each site
   is 4-6 lines of `const p = …; if (p instanceof Promise) p.catch(() =>
   {});`. A `tools/tools.ts`-level helper like
   `fireAndForgetPublish(bus, message)` would centralize the pattern, give
   you one place to adjust the swallow-vs-log policy, and let the tests
   spy on a single function. Not blocking but the diff has 7 sites today
   and grows linearly with every new fire-and-forget caller.

3. **`sanitizeToolArgs` integration is invisible from this PR's diff
   header.** The PR description mentions sanitization but the diff only
   shows tests asserting `[REDACTED]` — the actual `sanitizeToolArgs`
   import-and-call inside `message-bus.ts` is in the +13/−4 there but
   isn't shown in the truncated diff. Worth confirming the sanitization
   path also fires for the *successful* publish-then-log code path, not
   just the validation-error path. The `should sanitize sensitive data in
   debug logs` test at `message-bus.test.ts:200-222` does drive a
   successful publish (`type: TOOL_EXECUTION_SUCCESS`), so this is
   probably fine; just worth a manual eyeball on `message-bus.ts:`
   confirming `sanitizeToolArgs` is called both pre-emit (for debug log)
   and pre-throw (for error message).

4. **`rejects.toThrow(/\[REDACTED\]/)` followed by
   `rejects.not.toThrow(/secret-password/)` calls publish twice** at
   `message-bus.test.ts:181-187`. Each call is a fresh publish with the
   same invalid payload — works because the error path is deterministic,
   but it's two calls where one would do. Minor cleanliness:

   ```ts
   await expect(messageBus.publish(invalidMessage)).rejects.toThrow(
     expect.objectContaining({
       message: expect.stringMatching(/\[REDACTED\]/) &&
                expect.not.stringMatching(/secret-password/)
     })
   );
   ```

   or just capture the error once and assert on it.

5. **`MessageBusType.SUBAGENT_ACTIVITY` request-then-await test at
   `message-bus.test.ts:355-377`** uses a 10ms timeout for a guaranteed
   timeout test. On a busy CI runner this can be flaky — the request loop
   has to set up its listener, mint a correlation id, await, AND time-out
   all within 10ms before the test wall-clock advances. Bumping to 50-100ms
   is cheap and removes the flake risk.

6. **Doc-string on `publish()` should call out the new contract.** Future
   contributors approaching `MessageBus` for the first time will look at
   `publish()` and see it both emits on `'error'` AND throws — which is
   the right design but unconventional. A 2-line `/** … */` block above
   `publish` saying "throws after emitting `'error'` so awaiting callers
   observe failures; emit-only callers must `.catch()` the returned
   Promise" prevents the next caller from re-introducing
   `void messageBus.publish(…)`.

## Verdict rationale

Real bug fix (60-second hang reproduced, root cause identified, regression
test pins both the failure direction and the latency budget), defensive
hardening that addresses adjacent issues (unhandled-rejection crashes,
secret leakage in logs), and the call-site rewrites are uniform and
test-covered. Concerns are all stylistic / docs / test-stability. Ship it.
