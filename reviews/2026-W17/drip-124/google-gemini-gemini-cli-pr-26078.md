# google-gemini/gemini-cli #26078 — fix(cli): preserve Request headers in DevTools activity logger

- **PR**: [google-gemini/gemini-cli#26078](https://github.com/google-gemini/gemini-cli/pull/26078)
- **Head SHA**: `d683bbca`
- **Stats**: +69/-7 across 2 files

## Summary

Fixes the DevTools fetch interceptor in
`packages/cli/src/utils/activityLogger.ts:patchGlobalFetch` (around
`:302-330`) so it correctly preserves headers, method, and (for the
method/headers paths) request body when the caller passes a `Request`
object as the first argument to `fetch` (e.g. `fetch(new Request(url,
{headers, method, ...}))`) rather than splitting metadata into the
`init` parameter (e.g. `fetch(url, {headers, method, ...})`). The prior
implementation only read from `init`, so any caller using the `Request`
form (notably `ky` for update checks against private npm registries
that require auth) had its `Authorization` header silently stripped at
the interceptor and then rejected `403` upstream.

## Specific findings

- `activityLogger.ts:305-309` — new `inputMethod` extraction:
  `typeof input === 'object' && 'method' in input ? input.method :
  undefined`, then `const method = (init?.method || inputMethod ||
  'GET').toUpperCase()`. The precedence (`init.method` wins, falls
  back to `Request.method`, finally `'GET'`) matches the actual
  semantics of `new Request(input, init)` — when both are provided,
  `init` overrides — so the interceptor's reconstructed call passes
  through to `originalFetch` with the correct effective method.
- `:312-320` — same pattern for headers: extract `inputHeaders` from
  `Request.headers` first (so they're the base layer), then layer
  `init.headers` on top via `forEach((value, key) => headers.set(...))`
  — the `set` (not `append`) is correct because per-`init`-override
  semantics require replacement, not stacking. Then unconditionally
  `headers.set(ACTIVITY_ID_HEADER, id)` so the activity tracking
  header is always added regardless of caller form.
- `:323-326` — `newInit.method = method` is set explicitly. Without
  this line, callers passing a `Request` with non-GET method but no
  `init` would have `newInit.method` stay undefined and the
  reconstructed `originalFetch(input, newInit)` call would use the
  Request's original method via the `(input, init)` constructor's
  method-precedence rules. The explicit set is belt-and-suspenders —
  but worth keeping because the tests assert
  `expect(calledInit?.method).toBe('POST')` and rely on it being on
  `newInit`, not implicit from `input`.
- `:329-333` — body-extraction guard `const body = init?.body` is
  intentionally *not* extended to read `input.body` (Request bodies
  are streams that, once read, can't be re-read; pulling
  `await input.text()` to log the body would consume it before the
  real `originalFetch(input, newInit)` call). Correct conservative
  choice — body logging is best-effort and dropping it for
  Request-form callers is better than breaking the request.
- `activityLogger.test.ts:135-181` — new test
  `preserves headers and method from Request object when intercepting
  fetch`. Builds an actual `Request('https://api.example.com/data',
  {headers: {Authorization: 'Bearer test-token'}, method: 'POST'})`,
  calls `global.fetch(request)`, and asserts the mocked underlying
  fetch saw both the auth header *and* the activity-id header *and*
  the POST method on `calledInit`. Critically uses real `Request`
  + `Headers` objects rather than mocks — so it pins the real
  `Headers.get`/`Headers.has` semantic against the interceptor's
  reconstructed init. The `// @ts-expect-error` lines for poking
  `isInterceptionEnabled` are an unfortunate test smell but
  reasonable for the in-test-only reset.

## Nits / what's missing

- The `typeof input === 'object' && 'method' in input` guard at `:305`
  is defensive but `input` to `patchGlobalFetch`'s wrapper is typed as
  `RequestInfo | URL` — for `URL` instances `'method' in input`
  returns `false` so the check is harmless, but a more precise
  `input instanceof Request` would read better and tree-shake against
  the type-narrowing path.
- No test for the `URL` case (`fetch(new URL('...'))`) — should be
  covered by the existing string-URL test path but worth a one-line
  test to lock in that `URL` inputs don't accidentally match the
  `'method' in input` check via prototype pollution / future spec
  changes.
- The `Headers` layering pattern (init overrides Request) is not
  asserted — a test where `Request` has `Authorization: A` and `init`
  passes `Authorization: B` should assert the final request sees `B`,
  pinning the precedence semantic against a future "swap to forEach
  base order" regression.
- Comment block above `:305-318` would help — the
  init-overrides-Request precedence rule isn't obvious from the code
  and matches the WHATWG fetch spec. A two-line `// `
  reference would prevent drift.

## Verdict

**merge-after-nits** — fix is correct (closes the
`Authorization`-header-stripping bug for `ky`-style Request callers),
test coverage pins the headers + method preservation in one
combined real-Request test, and the body-extraction guard is
correctly *not* extended to avoid consuming the Request body. Three
testing/documentation nits but the actual fix is good to ship.
