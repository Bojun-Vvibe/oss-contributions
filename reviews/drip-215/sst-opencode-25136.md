# Review: sst/opencode#25136 — Preserve workspace context in session HTTP routes

- PR: https://github.com/sst/opencode/pull/25136
- Merge SHA: `feeebbe7d4ebf1fdaa74cc1aa63e11cb78b97f8a`
- Files: `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts` (+~140/-~240), and 3 supporting files
- Verdict: **merge-after-nits**

## What it does

The keystone PR of Kit's three-PR HTTP-handler-consolidation series
(#25136 session, #25138 PTY, #25139 workspace). Lifts seven service
references — `Session.Service`, `SessionShare.Service`, `SessionPrompt.Service`,
`SessionRevert.Service`, `SessionCompaction.Service`, `SessionRunState.Service`,
`Agent.Service`, `Permission.Service` — out of per-handler
`Effect.gen(function* () { const svc = yield* X.Service; ... })` blocks
and into a single top-of-group `yield*` cascade at `session.ts:55-62`. Then
strips the `Instance.restore(instance, () => AppRuntime.runPromise(...
.pipe(Effect.provide(SomeService.defaultLayer))))` wrapper from ~14 handler
bodies (`update`, `fork`, `abort`, `init`, `share`, `unshare`, `summarize`,
`prompt`, `command`, `shell`, `revert`, `unrevert`, `permissionRespond`,
`remove`, `create`).

For the two streaming handlers (`prompt`, `promptAsync`), substitutes
`bridge.promise(...)` / `bridge.fork(...)` from `EffectBridge.make()` for
the legacy `Instance.restore` + `AppRuntime.runPromise` + `.catch` pattern.

## Notable changes

- `session.ts:55-62` — eight services lifted to the construction-site `yield*`
  cascade. The ordering matters because `Effect.gen` evaluates eagerly;
  hoisting these means a missing layer fails at handler-group-construction
  time (so the server crashes at boot) rather than at first-request time
  (where it would manifest as a 500 from a specific endpoint). This is the
  right tradeoff for a production HTTP surface.
- `session.ts:245` — `summarize` collapses from a 30-line `Instance.restore`
  + `AppRuntime.runPromise` + 5x `Effect.provide(...)` cascade to 11 lines
  of direct service calls. The pre-state was building a *new* runtime per
  request and providing five service layers into it; the post-state reuses
  the already-provided layers from the route's outer `mergeAll`.
- `session.ts:264-275` (`prompt` streaming handler) — `Instance.restore` →
  `AppRuntime.runPromise` chain replaced with `EffectBridge.make()` +
  `bridge.promise(promptSvc.prompt(...))`. The `EffectBridge` pattern was
  introduced for exactly this case: bridge an Effect into the
  `Stream.fromEffect(Effect.promise(() => ...))` shape required by
  `HttpServerResponse.stream`. The promise wrapper around the bridge is
  preserved because the outer `Stream.fromEffect` API takes a Promise-based
  shape.
- `session.ts:286-307` (`promptAsync`) — fire-and-forget shape via
  `bridge.fork(promptSvc.prompt(...).pipe(Effect.catchCause(error => ...))).`
  The error handler now uses `Effect.catchCause` instead of `.catch` on the
  Promise, which is the right call: it captures defects (panics) as well as
  failures, where the prior `.catch` only caught Promise rejections. So this
  is technically a behaviour improvement under fire-and-forget semantics —
  a panic in `promptSvc.prompt` would previously have been an unhandled
  promise rejection, but now publishes a `Session.Event.Error` and logs.
- The error-cause shape changes from `error instanceof Error ? error.message
  : String(error)` to bare `String(error)`. For `Effect.catchCause`, the
  argument is a `Cause<E>`, not an `Error`; `String(cause)` produces a
  pretty-printed cause tree (failure chain + defects + interrupts). This is
  *more* informative than the prior `Error.message`, but the resulting
  `NamedError.Unknown` payload will be longer — worth confirming downstream
  serializers (Bus consumers, log shippers) can handle multi-line strings.
- `session.ts:1` — `EffectBridge` import added, replacing `AppRuntime`
  import (which is no longer needed since the handlers don't construct
  per-request runtimes).

## Reasoning

This is the load-bearing change in the multi-PR consolidation. The
`Instance.restore(instance, () => AppRuntime.runPromise(...))` pattern was
a transitional shape from when the HTTP layer wasn't fully Effect-native —
each handler was constructing a fresh runtime with the right service
layers, executing the work in it, and binding instance scope around the
boundary. Now that `routes` is built as a single `Layer.mergeAll` with all
the service layers provided once at route-mount time (visible in the
matching change to `server.ts` in the related PRs), the per-handler runtime
construction is pure overhead — both in CPU (layer building per request)
and in cognitive load (every handler had a 5-7 line wrapper).

The `EffectBridge` choice for the streaming handlers is correct because
`HttpServerResponse.stream` takes a `Stream` built from a `Promise`, which
forces *some* Effect→Promise bridge somewhere. The bridge service is
explicitly that boundary and is Effect-context-aware (preserves fiber
context, interruption signals, etc.), which `AppRuntime.runPromise` was
not — so this is also a correctness improvement for cancellation behaviour.

The `Effect.catchCause` upgrade in `promptAsync` is the kind of subtle win
that earns this PR its grade: the prior `.catch(error => ...)` on the
Promise would silently swallow Effect *defects* (uncaught panics inside the
Effect) because `runPromise` translates those into rejections at a layer
where the cause is already opaque. `Effect.catchCause` operates at the
Effect layer where defects are first-class and can be logged with full
fiber stack and interruption history. This is invisible until something
actually panics, then it's the difference between debuggable and not.

## Nits (non-blocking)

1. `session.ts:301` — `String(error)` where `error` is now a `Cause<E>` will
   produce multi-line output that downstream Bus consumers may not expect.
   Consider `Cause.pretty(error)` or `Cause.squash(error).message ??
   String(error)` so the `NamedError.Unknown.message` field stays
   single-line. Mostly a UX consideration for error-aggregation dashboards.
2. `session.ts:296` — `Effect.sync(() => bridge.fork(...))` wraps a
   side-effecting `bridge.fork` in `Effect.sync` to return `void`; this is
   correct but slightly awkward. If `bridge.fork` returns the fiber, the
   wrapper could be `Effect.fork(promptSvc.prompt(...).pipe(...))` directly
   — but only if the bridge's lifecycle semantics allow that.
3. The eight `yield* X.Service` hoists at `:55-62` lose the per-handler
   "you only depend on what you yield" property. If `permissionRespond`
   doesn't actually use `compactSvc` it will still fail at boot if
   `SessionCompaction.defaultLayer` isn't provided. That's fine for a
   tightly-coupled handler group but worth noting if the group ever splits.
4. The pre-state had per-handler `log.info("revert", ctx.payload)` at the
   top of `revert`; the new state doesn't. Confirm the log was redundant
   (probably yes, since `Effect.fn("SessionHttpApi.revert")` already adds
   a tracer span).
5. The `as unknown as SessionPrompt.PromptInput` casts in `prompt` /
   `promptAsync` survive the refactor unchanged. They were a workaround for
   payload-schema-vs-input-schema drift; would be nice to fix but out of
   scope here.
