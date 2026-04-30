---
pr-url: https://github.com/sst/opencode/pull/25163
sha: 1900e648dc23
verdict: merge-after-nits
---

# Fix HttpApi web UI fallback

Restores the web-UI fallback mount under the experimental HttpApi backend by adding a catch-all `HttpRouter.add("*", "/*", ...)` route at `packages/opencode/src/server/routes/instance/httpapi/server.ts:124-135` that lazy-loads `UIRoutes()` and proxies the request through it. The keystone shape: when the inbound `request.source` is already a Web `Request`, it's passed straight through; otherwise a synthetic `new Request(new URL(request.originalUrl, "http://localhost"), { method, headers })` is constructed, the response is mapped via `HttpServerResponse.fromWeb`. The new `uiRoute` is appended to `Layer.mergeAll(rootApiRoutes, instanceRoutes, uiRoute)` at `:139` so existing API routes match first and the UI only catches genuinely unmatched paths — pinned by the second test at `httpapi-ui.test.ts:34-44` which monkey-patches `globalThis.fetch` to throw and asserts `/session/nope` returns 404, not 500.

Nits: (1) the synthetic-`Request` path drops the request *body* — fine for the GET-only UI fallback today, but if `UIRoutes` ever serves POST endpoints (asset uploads, telemetry beacons) this will silently 400; worth a comment at `:131` documenting the body-less assumption. (2) `lazy(() => UIRoutes())` is invoked per route registration, but the closure inside `Effect.promise(async () => uiRoutes().fetch(...))` calls `uiRoutes()` once per request — `lazy` here memoizes correctly, but the double-paren shape is read-stumbling; rename the binding `getUiRoutes` for clarity. (3) the test at `:24` asserts `proxiedUrl === "https://app.opencode.ai/"` — that hard-pins the upstream UI host into a unit test, so any future env-driven host swap will break this test before it breaks production.

## what I learned
When you bolt a catch-all route onto a router that uses positional matching, the *order* of the `Layer.mergeAll` call is the contract — and the test that nails the ordering (`/session/nope` → 404, not UI proxy) is more load-bearing than the test that nails the happy path. Always write the negative test before declaring the wildcard route safe.
