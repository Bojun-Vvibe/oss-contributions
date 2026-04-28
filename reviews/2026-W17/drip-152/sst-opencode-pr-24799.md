# sst/opencode#24799 — refactor(httpapi): fork server startup by flag

- **Repo:** [sst/opencode](https://github.com/sst/opencode)
- **PR:** [#24799](https://github.com/sst/opencode/pull/24799)
- **Head SHA:** `887abc13d6e924d9bd5250847c205f175c867085`
- **Size:** +190 / -371 across 23 files (server adapters, generate.ts, plugin index, all `httpapi-*.test.ts` harnesses)
- **State:** MERGED

## Context

Up to now the experimental Effect HttpApi surface lived as a *shadow*
route layer inside the legacy Hono app — the Hono router still owned
the request, but for any path matching a registered HttpApi route
the request was forwarded into Effect's handler stack. That shape was
fine while the HttpApi surface was small and incomplete, but it had
two real costs: (1) every request paid the cost of Hono's path
matching even when destined for Effect, and (2) reasoning about which
side actually owns a route required reading both routers. The PR
hard-forks at server startup: when `OPENCODE_EXPERIMENTAL_HTTPAPI` is
set, the outer server runs the Effect handler stack directly with no
Hono in the path; otherwise legacy Hono runs as before.

## Design analysis

Five layers move:

1. **Adapter contract widening** at `adapter.ts:5` (and re-exports
   into `adapter.bun.ts:11-44`, `adapter.node.ts:6-44`). The adapter
   gains a new `createFetch(app)` factory alongside the existing
   `create(hono)` factory. `FetchApp` is the minimum interface
   (just `fetch: (req) => Response`) that both Hono apps and Effect's
   `HttpApiBuilder.toFetch` outputs satisfy. Right shape — the
   adapter was previously coupled to Hono's full surface (websockets,
   middleware), the new `FetchApp` interface only needs what the
   listener actually calls.

2. **Per-runtime listener extraction**. `adapter.bun.ts:5-30` and
   `adapter.node.ts:5-44` both factor a shared `listen(app, opts,
   ...optionalRuntimeBits)` function that the existing `create()` and
   the new `createFetch()` both call into. The Bun version's
   `listen(app, opts, websocket?)` carries the optional websocket via
   a third arg and conditionally selects `Bun.serve({...websocket})`
   or the no-websocket overload at `:6-12` — the optional websocket
   path is load-bearing because the Effect HttpApi doesn't currently
   wire websockets, so the no-websocket branch is the path the new
   `createFetch()` will hit. Node version does the same with an
   `inject?: (server) => void` callback at `:6` for the `setupNodeWs`
   integration on the legacy side.

3. **Server selection** at `server.ts:28-42` reads
   `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` and forks: experimental path
   builds via `HttpApiBuilder.toWebHandler(PublicApi)` and wraps in
   `adapter.createFetch(...)`, legacy path builds the Hono app and
   wraps in `adapter.create(...)`. The fork happens at *server-build*
   time not per-request — so a flag flip requires server restart, not
   just request re-classification. That's the right tradeoff for an
   experimental gate (the per-request fork is more expensive and
   harder to reason about), but the implication needs a startup-log
   line so operators flipping the flag understand why they need to
   restart.

4. **Plugin layer cleanup** at `plugin/index.ts:130`. The
   `Server.Default()` call is no longer async (was `(await
   Server.Default()).app.fetch(...)` — now `Server.Default().app.fetch(...)`).
   This is a knock-on from the restructure: server building no longer
   returns a Promise because the Effect handler-stack build is sync.
   Subtle but load-bearing — any plugin or test that previously
   `await`-ed the Server.Default call still works (await on a
   non-Promise returns the value), but anyone who *typed* the result
   as `Promise<Server>` will now see a type error. Worth a TS-strict
   build against a few real plugins to confirm.

5. **Massive test conversion** across `httpapi-*.test.ts` — every
   file drops the `await using server = Server.create()` boilerplate
   in favor of `Server.Default().app` directly, and the
   `httpapi-bridge.test.ts` (which used to assert legacy-vs-Effect
   route parity) loses 88 lines because the bridge it was testing
   no longer exists. The remaining 3-line skeleton at the file-end
   keeps the parity-route-name assertion that `OpenApi.fromApi(PublicApi)`
   declares all the same routes as legacy — good, that's the contract
   even after the bridge is gone.

The CLI generate command at `cli/cmd/generate.ts:8-26` is the most
quietly useful addition: a new `--httpapi` flag that flips the OpenAPI
*generator* to read from `OpenApi.fromApi(PublicApi)` instead of
`Server.openapi()`, so the SDK build can be regenerated against the
Effect surface for parity verification without restarting any server.
The PR body's claim that "default Hono source produced no generated
diff" is the right validation — the SDK regeneration with the legacy
flag must be byte-identical to confirm zero behavior change for
non-experimental users.

## Risks / nits

1. **No startup log for which fork was selected.** When operators flip
   `OPENCODE_EXPERIMENTAL_HTTPAPI`, there's no console signal that
   server is running on the new path. A single `info!` log at
   `server.ts:28-29` ("starting server with experimental HttpApi
   surface" / "starting server with legacy Hono surface") makes
   support tickets dramatically easier. Critical for an experimental
   gate.

2. **`createFetch` adapter has no error-recovery semantics for handler
   crashes.** The legacy Hono path benefits from Hono's built-in
   error middleware that converts uncaught throws to 500s; the new
   `HttpApiBuilder.toWebHandler` path inherits Effect's error
   handling, which on uncaught defects throws *out* of the fetch
   call. A defect in any HttpApi handler will crash the entire
   request handler in a way that legacy never did. Worth a
   `Effect.catchAllDefect` wrapper at the toWebHandler boundary
   converting defects to a 500 response with logging.

3. **Plugin layer's `Server.Default().app.fetch` change is silently
   breaking for any plugin code that captured the Promise.** The
   refactor at `plugin/index.ts:130` removes the `await` because
   `Server.Default()` is now sync, but plugins consuming the public
   `Server` type from `@opencode/server` will see a TypeScript error
   on the next typecheck. This is fine for first-party but warrants
   a `BREAKING:` callout in release notes for plugin authors.

4. **Massive test diff is correct but hard to review.** 18 test files
   each lose 4-8 lines of identical boilerplate. The shape is fine
   but the right-cadence here would have been to extract a shared
   `setupHttpApiTest()` helper *first* in a precursor PR, then have
   this PR's diff be a single-line change per test. Hindsight, but
   the all-files-at-once shape makes it impossible to spot
   inconsistencies between files (e.g., did `httpapi-tui.test.ts`
   drop a relevant `Flag.reset` that the others kept?).

## Verdict

**merge-after-nits.** The architectural shape is right: hard-fork at
startup is simpler than per-request shadow routing, the adapter
widening to support both `Hono` and `FetchApp` is well-scoped, and
the parity test surface is preserved. Pre-merge asks: nit 1 (startup
log) and nit 2 (defect recovery at the toWebHandler boundary). Nit 3
needs a release-notes callout. Nit 4 is a process retrospective.

## What I learned

- "Hard-fork at the server-build seam" is almost always cleaner than
  "shadow-route at the request seam" when migrating between two
  parallel implementations of the same surface — the cost is a
  required restart on flag flip, the win is dramatically simpler
  reasoning about which side owns a request.
- Effect's `HttpApiBuilder.toWebHandler` converts an HttpApi
  declaration into a `(Request) => Promise<Response>` function that
  satisfies the same `FetchApp` interface as `Hono.fetch`. That
  shared minimum interface is the right boundary between the
  framework-agnostic listener and the framework-specific app.
- When refactoring removes an `await` (because some upstream became
  sync), callers that previously typed the result as `Promise<T>`
  will see TypeScript errors even though their `await` still works
  — worth a release-notes BREAKING callout for any external consumer.
