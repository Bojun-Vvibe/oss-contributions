# sst/opencode#24835 — fix(httpapi): wire global and control handlers

- **Repo:** [sst/opencode](https://github.com/sst/opencode)
- **PR:** [#24835](https://github.com/sst/opencode/pull/24835)
- **Head SHA:** `6b5f9bdd304d13923312e05037d4514cf92cfc5d`
- **Size:** +448 / -207 across 18 files
- **State:** OPEN

## Context

Companion to #24836 (the SDK-level test that exercises these routes). This PR
finishes the Effect-HttpApi parity work that drip-148 #24827 started: it (a)
adds runtime handlers for the `global` and `control` API groups that were
previously only wire-format declarations, and (b) restructures the route
composition so global/control routes live outside the per-instance auth
middleware while instance routes still pass through it.

## Design analysis — three things at once, each correct

### 1. New handlers for `global` and `control` (`global.ts:+158`, `control.ts:+30`)

`controlHandlers` (`control.ts:73-99`) wires the three control-plane endpoints
(`authSet`, `authRemove`, `log`). The `Effect.fn(...)` naming
(`"ControlHttpApi.authSet"`) gives Otel spans a useful name, and
`Effect.orDie` on the auth calls is the right policy choice — control-plane
mutations that fail at the auth-store layer should crash the request rather
than be coerced into a typed-error envelope, because callers can't recover
from "auth.set itself raised."

`globalHandlers` is the meatier piece. The `eventResponse()` helper at
`global.ts:117-160` is the SSE construction:

```ts
const cleanup = () => {
  if (done) return
  done = true
  if (heartbeat) clearInterval(heartbeat)
  unsubscribe()
  log.info("global event disconnected")
}
```

The `done` guard is correctly checked in both the `cancel` callback path *and*
inside `write()` (lines 132-138) where any controller-throw triggers cleanup.
That's the right shape — `controller.enqueue` after the consumer drops will
throw, so wrapping it in try/catch + cleanup is the canonical Web Streams
pattern for SSE.

The `upgradeRaw` handler (`global.ts:218-237`) is interesting: it shadows the
typed `upgrade` handler with a `handleRaw` variant that does manual
`Schema.decodeUnknownEffect` + custom 400 envelope. The reason isn't documented
in the diff but is presumably so the response body contract is `{success,
error}` JSON rather than the Effect HttpApi default error shape — worth a
one-liner comment.

### 2. Layer simplification — `Layer.unwrap` → `HttpApiBuilder.group` directly

The repeated rewrite across 11 group files (config.ts, experimental.ts,
file.ts, instance.ts, mcp.ts, permission.ts, project.ts, provider.ts, pty.ts,
question.ts, session.ts, sync.ts, tui.ts, workspace.ts) is mechanically
identical:

```ts
// before:
export const fooHandlers = Layer.unwrap(
  Effect.gen(function* () {
    /* … */
    return HttpApiBuilder.group(FooApi, "foo", (handlers) =>
      handlers.handle("a", a).handle("b", b),
    )
  }),
).pipe(Layer.provide(Foo.defaultLayer))

// after:
export const fooHandlers = HttpApiBuilder.group(FooApi, "foo", (handlers) =>
  Effect.gen(function* () {
    /* … */
    return handlers.handle("a", a).handle("b", b)
  }),
)
```

The behavioral difference is subtle but real: the `Layer.provide(Foo.defaultLayer)`
calls are no longer per-group; they've all been hoisted into the single
`Layer.provide(...)` chain at `server.ts:147-167`. That's the right move
because (a) the same service can now be shared across groups without rebuilding
its layer per group, and (b) you read a single ordered list to know what's
provisioned. The removed `Layer.unwrap(...)` indirection was load-bearing only
because each group needed to inject its own dependencies — once that
responsibility is centralized, the unwrap dance becomes dead.

### 3. Routing split — global/control outside auth, instance inside

`server.ts:99-130` is the new composition:

```ts
const controlRoutes = HttpApiBuilder.layer(ControlApi).pipe(Layer.provide(controlHandlers))
const globalRoutes  = HttpApiBuilder.layer(GlobalApi).pipe(Layer.provide(globalHandlers))
const instanceApiRoutes = Layer.mergeAll(/* … 12 groups … */)

const instanceRoutes = Layer.mergeAll(eventRoute, ptyConnectRoute, instanceApiRoutes).pipe(
  Layer.provide(authorizationLayer),
  Layer.provide(instance),
)

export const routes = Layer.mergeAll(controlRoutes, globalRoutes, instanceRoutes).pipe(/* shared deps */)
```

This is the *correct* boundary: `/health`, `/event` (the global SSE), `/upgrade`
must work without an instance/auth context (they're how the host bootstraps,
and how a freshly-launched server reports liveness). `auth.set / auth.remove`
must also work without instance context because they're how a new instance
gets its first credentials. Putting these inside `instance` middleware would
have been a chicken-and-egg deadlock.

## Risks / nits

1. **`server.ts:147-167` — provider order matters and isn't documented.** The
   second `.pipe()` chain (`SessionRunState`, `SessionStatus`, `SessionSummary`,
   `Skill`, `Todo`, `ToolRegistry`, `Vcs`, `Worktree`) is split from the first
   for what looks like a TypeScript compiler limit (overly long pipe chain
   inferring `never`). A `// note: split into two .pipe(...) chains to keep TS
   inference happy` would save the next reader 20 minutes.
2. **`global.ts:eventResponse()` — no `last-event-id` handling.** SSE clients
   on reconnect send `Last-Event-ID`; this handler ignores it and starts the
   stream from "now." For `server.connected` / `server.heartbeat` / `global.disposed`
   that's fine (no replay semantics). If anyone ever pipes session events
   through `GlobalBus` the absence will quietly cause missed-event windows on
   reconnect. Worth a comment that this stream is intentionally non-replayable.
3. **`global.ts:upgradeRaw` — duplicated decode logic.** The handler decodes
   `GlobalUpgradeInput` manually then calls into `upgrade(...)` which the
   typed handler also wraps. If `GlobalUpgradeInput` changes, both paths need
   to stay in sync. Extracting the decode + dispatch into a single
   `runUpgrade(payload)` helper would prevent that drift.
4. **`control.ts:authSet` returns `true`.** `Effect.orDie` means the only way
   this returns is success, so `true` is fine — but a typed `{success: true}`
   shape would be more consistent with `globalHandlers.dispose` /
   `globalHandlers.upgrade` which both return structured envelopes. Pure
   nit, established convention call.

## Verdict

**merge-after-nits.** The structural work is correct: handlers for the two
missing groups, layer dedup across 11 files, routing split that puts
global/control outside auth. The nits are documentation-grade — comments
about the split-pipe rationale, the non-replayable SSE intent, and either a
shared decode helper for `upgradeRaw` or a comment justifying the duplication.

## What I learned

- When every Effect group needs the same set of `Layer.provide`s, hoisting
  them out of per-group `Layer.unwrap` indirections collapses the boilerplate
  and makes the dep set readable from one place.
- `HttpApiBuilder.group` is built to take an `Effect`-returning callback —
  the older `Layer.unwrap(Effect.gen(... return HttpApiBuilder.group(...)))`
  pattern was awkward because the same thing was being expressed twice.
- Splitting middleware boundaries by *bootstrap-vs-instance* responsibility
  (rather than by route prefix) is the correct mental model for any
  multi-tenant local server.
