# Review: sst/opencode#25139 — Use workspace service in HTTP routes

- PR: https://github.com/sst/opencode/pull/25139
- Merge SHA: `19271fca2d2bcbd45bfc39f73c441db0ec9b94f9`
- Files: `packages/opencode/src/server/routes/instance/httpapi/handlers/workspace.ts` (+18/-23), `packages/opencode/src/server/routes/instance/httpapi/server.ts` (+3/-1)
- Verdict: **merge-as-is**

## What it does

Migrates the `workspace` HTTP handler group from the legacy
`Instance.restore(instance, () => Workspace.create(...))` Promise-bridge
pattern to the canonical Effect service pattern: `const workspace = yield*
Workspace.Service` at handler-group construction, then `yield* workspace.X(...)`
at each call site. Adds `Workspace.defaultLayer` to the `routes` `mergeAll` at
`server.ts:145` so the service is provided.

## Notable changes

- `workspace.ts:11` — `const workspace = yield* Workspace.Service` lifted to
  the top of the `Effect.gen` block, consistent with how the neighbouring
  `session` handler group already structured itself in #25136 and how the
  `pty` group did in #25138 (drip-213). Kit's stack is converging the entire
  HTTP surface on this shape.
- `workspace.ts:21` — `list` drops `Workspace.list((yield*
  InstanceState.context).project)` (which called the static `Workspace.list`
  exported from `@/control-plane/workspace`) for `yield* workspace.list(...)`.
  Same for `status` at `:32-35`, `remove` at `:38-39`, `create` at `:23-30`,
  and `sessionRestore` at `:41-49`.
- `workspace.ts:25-30` and `:46-49` — `create` and `sessionRestore` no longer
  wrap with `Effect.promise(() => Instance.restore(instance, () =>
  Workspace.create(...)))`. The `Instance.restore` was doing two things:
  binding the Effect into instance scope so the worker process knew which
  project to operate on, and converting the unbounded Promise to an
  Effect-aware error path. The service-shape replacement gets the first for
  free (the service is constructed inside the instance-scoped Effect runtime)
  and replaces the second with `.pipe(Effect.mapError(() => new
  HttpApiError.BadRequest({})))`, which collapses any internal failure to the
  HTTP 400 the route was already returning.
- `workspace.ts:38` — `remove` no longer needs the `Effect.mapError`
  redirect. Looking at the pre-state, the `Instance.restore` wrapper would
  have surfaced any thrown error as an `Effect.die` via `Effect.promise`'s
  contract; the new direct `yield*` exposes the typed error channel from the
  service. If that channel includes anything beyond the existing 404 / 400
  shapes, the new shape will let it propagate uncaught — worth confirming
  the `workspace.remove` error type only includes `Workspace.NotFound`
  variants that are already mapped by the surrounding `mapNotFound` helper
  (or its absence here is a deliberate "404 means idempotent succeed" call).
- `server.ts:34, :145` — `Workspace.defaultLayer` added to imports and to the
  `Layer.mergeAll` provision list, slotted alphabetically between `Vcs` and
  `Worktree`.

## Reasoning

This is part of a multi-PR consolidation (drip-213's #25138 PTY, #25139 here,
#25136 session — all by Kit on the same day) that strips the Effect↔Promise
round-trips that were a transitional shape during the HttpApiBuilder
migration. The pattern they're replacing — `Instance.restore(instance, () =>
SomeService.use(...).pipe(Effect.provide(SomeService.defaultLayer))` inside
`AppRuntime.runPromise` inside `Effect.promise` — is fundamentally a
project-instance binding hack from before the routes mounted into an
already-instance-scoped runtime. Now that `routes` is built via
`Layer.mergeAll(rootApiRoutes, instanceRoutes).pipe(Layer.provide(...))` in
`server.ts:138`, the handlers run inside that layer's provided services and
no longer need to re-establish instance scope per call.

The `Effect.mapError(() => new HttpApiError.BadRequest({}))` pattern at
`create` and `sessionRestore` is the right shape because both operations
have multi-cause failure modes (workspace name collision, restore-on-deleted
session, etc.) that all collapse to a 400 from the client's perspective. The
empty `{}` means no error body — consistent with what `Effect.promise`'s
`Effect.die` was producing previously (an opaque 500 with no detail), so
this is actually a slight improvement: a bare 400 is more honest than an
opaque 500 for client-input errors.

## Nits (non-blocking)

1. `workspace.ts:38` — `remove` lost its error mapping. If the underlying
   `workspace.remove` can fail with anything other than the typed `NotFound`
   that other handlers translate via `mapNotFound`, this will become a 500.
   A `.pipe(Effect.mapError(() => new HttpApiError.BadRequest({})))` for
   parity with `create`/`sessionRestore` would be defensible, or
   alternatively `mapNotFound` if `NotFound` is the only failure mode.
2. `HttpApiError.BadRequest({})` is empty — would be nicer to surface the
   underlying Effect error message via `BadRequest({ message: error.message })`
   (if the schema allows) so clients get diagnostics. But that's a follow-up
   pattern that all three of Kit's PRs share, not specific to this one.
3. The `adaptors` handler at `:13-16` is still using `Effect.promise(() =>
   listAdaptors(instance.project.id))` — that one isn't an Effect-native
   service yet so it's fine to leave, but worth a `// TODO` if it's on the
   migration roadmap.
