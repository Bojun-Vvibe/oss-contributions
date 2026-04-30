---
pr-url: https://github.com/sst/opencode/pull/25179
sha: cb4a372c74d3
verdict: merge-after-nits
---

# Avoid request-time HttpApi layer provisioning

Plumbing refactor that moves `HttpClient.HttpClient` and `HttpRouter.HttpRouter` from per-request `Effect.provide(FetchHttpClient.layer)` calls into proper service-acquisition at layer-construction time, plus flips the `eventRoute` and `uiRoute` definitions from the `Layer.effectDiscard(... yield* HttpRouter.HttpRouter ... yield* router.add(...))` shape to the `HttpRouter.use((router) => ...)` shape.

The load-bearing change is at `httpapi/middleware/proxy.ts:67-91`: `http()` grows a leading `client: HttpClient.HttpClient` parameter, the inner body switches from `HttpClient.execute(...)` (which used the ambiently-provided client) to `client.execute(...)`, and the trailing `.pipe(Effect.provide(FetchHttpClient.layer), ...)` loses the `Effect.provide(FetchHttpClient.layer)` arm. The same pattern fans out through `workspace-routing.ts:97-225` — `proxyRemote`, `routeWorkspace`, `routeHttpApiWorkspace` all grow a leading `client` parameter, and both `workspaceRoutingLayer` and `workspaceRouterMiddleware` add a `const client = yield* HttpClient.HttpClient` at layer-construction time so the client is acquired once at startup, not once per request.

The motivation is correct and important: every request previously paid the `Effect.provide(FetchHttpClient.layer)` cost, which under Effect builds a fresh runtime sub-context per call. For an SSE route like `/event` which holds the request open for the connection lifetime, that's a one-time cost; for a workspace-proxy route firing on every UI fetch, that's a measurable per-request allocation that shows up in P99 latency. Moving the client acquisition to the layer boundary is the textbook fix.

The `eventRoute` rewrite at `event.ts:39-72` is the more-subtle half of the change — flipping from `HttpRouter.add("GET", EventPaths.event, Effect.gen(...).pipe(Effect.provide(Bus.layer)))` to `HttpRouter.use((router) => Effect.gen(function* () { const bus = yield* Bus.Service; yield* router.add("GET", EventPaths.event, Effect.gen(...)) }))` lifts the `Bus.Service` acquisition out of the per-connection effect into the router-construction effect. The `bus.subscribeAll()` call still happens per-request (correct — each SSE connection needs its own subscription), but the `Bus.Service` *resolution* happens once at layer-build. Same pattern as the HttpClient change, just applied to a different service.

Three real nits:

(1) The PR doesn't include any benchmark or before/after timing for the per-request hot path. The motivation ("Avoid request-time HttpApi layer provisioning") implies a measurable win but the diff lands the change without a number. A 3-line `bench/proxy.bench.ts` (or even a one-line PR-description note like "P50 proxy latency drops from X ms to Y ms on synthetic workload") would let the next reader confirm the perf claim without re-running the experiment.

(2) The new `client: HttpClient.HttpClient` parameter is the *first* positional in `proxyRemote`, `routeWorkspace`, `routeHttpApiWorkspace`, and `http()`. Five call-site updates in this PR are all mechanical — but the next person adding a sixth caller has no compile-time hint that `client` should come from `yield* HttpClient.HttpClient` at the layer boundary versus being constructed ad-hoc. A brief docstring on each function naming the contract ("client must be the layer-resolved HttpClient, not a per-call construction") would help.

(3) The `eventRoute` flip removes the `Effect.provide(Bus.layer)` arm at `:73` but the file's import of `Bus.layer` is still in scope (or was — diff doesn't show imports). Worth a quick grep to confirm no dead imports remain, since dead-import drift is exactly the kind of thing this kind of refactor accumulates over multiple passes.

No security or correctness concerns. Layer-time service acquisition is the idiomatic Effect pattern and this PR is straightforwardly applying it to the request-hot-path services. Ready to merge once the perf claim is grounded.

## what I learned
"Provide the layer at the call site" is the lazy default for `Effect`-based service injection and is almost always wrong on a request-hot path, because layer-provisioning is *not* free — every call constructs a sub-runtime, even for stateless services like `FetchHttpClient`. The correct pattern is "acquire the service at layer-construction time, thread it as a parameter to the per-request handler" — which is what this PR does. The trade-off is that per-request handlers grow a parameter for every layer-resolved service they need, but that's the right cost: it makes the dependency explicit at the function signature, and it pays the layer-provisioning cost once at startup instead of N times at request time.
