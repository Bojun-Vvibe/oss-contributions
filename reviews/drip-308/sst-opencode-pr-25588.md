# sst/opencode #25588 — fix(httpapi): add basic auth challenge for browser login

- PR: https://github.com/sst/opencode/pull/25588
- Author: OpeOginni
- Head SHA: `c50510619bdebc239ecbde7cd4fbbdd8916c0c21`
- Updated: 2026-05-03T12:50:19Z

## Summary
Adds a `WWW-Authenticate: Basic realm="Secure Area"` response header on 401 from the experimental HttpApi authorization middleware so browsers will surface the native basic-auth login dialog. Tiny change with a matching test update.

## Observations
- `packages/opencode/src/server/routes/instance/httpapi/middleware/authorization.ts` line ~7 introduces `const WWW_AUTHENTICATE = "Basic realm=\"Secure Area\""` as a module-level constant. Reasonable, but since this header is emitted unconditionally on 401, sites that intentionally use bearer/token auth via `auth_token` query param will *also* see a basic-auth challenge after a failure — that may surprise non-browser clients. Consider gating the `www-authenticate` header on a basic-auth-supporting credential mode rather than emitting it on every unauthorized response.
- Same file, `validateRawCredential` (around line ~85): the previous `Effect.succeed(HttpServerResponse.empty({ status: UNAUTHORIZED }))` is now wrapped with a `headers` map. Functionally fine; preserves the empty body so no behavior change beyond the header.
- Test addition in `packages/opencode/test/server/httpapi-ui.test.ts` line ~204 (`expect(response.headers.get("www-authenticate")).toBe('Basic realm="Secure Area"')`) is a good regression anchor — but it only covers the UI fallback path. There is no companion test for the API path that hits the same `validateRawCredential` branch; worth adding to lock the contract on both surfaces.
- Realm string `"Secure Area"` is generic. Consider `"opencode"` so the browser's saved-credentials dropdown is namespaced sensibly.

## Verdict
`merge-after-nits`
