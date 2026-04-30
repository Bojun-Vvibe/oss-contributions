# Review: sst/opencode#25152 — Refactor workspace service boundaries

- PR: https://github.com/sst/opencode/pull/25152
- Head SHA: `e3cde1f749f3e71d5e9cdf0628bff529767d0803`
- Author: kitlangton
- Files: 14 changed (+421/-405)
- Merged: 2026-04-30

## Stated purpose
Tear down the legacy `Workspace` runtime façade — a hand-rolled `makeRuntime(Service, defaultLayer)`-derived `runPromise`/`runSync` pair plus seven `fn(...)`-wrapped exports (`create`, `sessionRestore`, `get`, `remove`, `status`, `isSyncing`, `waitForSync`, `startWorkspaceSyncing`) — and require all server boundaries to access `Workspace.Service` through the proper Effect provision chain. This is the keystone that lets PRs #25136/#25138/#25139 actually land cleanly: those refactored handler groups can only `yield* Workspace.Service` if there's a workspace-routing middleware providing it.

## What actually changed
1. **`control-plane/workspace.ts:855-895`** — 40-line block deleted: the `runPromise/runSync` pair plus all eight thin top-level wrappers. Module no longer carries its own runtime; consumers must obtain `Workspace.Service` from context.
2. **`server/fence.ts:57-78`** — `wait(workspaceID, state, signal)` is split into a new `waitEffect(...)` (Effect-returning, uses `Workspace.Service.use((workspace) => workspace.waitForSync(...))`) plus a thin async `wait(...)` shim that does `AppRuntime.runPromise(waitEffect(...))`. The Hono-side `FenceMiddleware` keeps its async shape — boundary preserved, internals migrated.
3. **`server/proxy.ts:73-86`** — `httpEffect` no longer calls the now-deleted `Workspace.isSyncing(workspaceID)` synchronously *outside* the `Effect.gen`; it's been pulled inside the generator and accessed via `yield* Workspace.Service.use((workspace) => workspace.isSyncing(workspaceID))`, with the early-return 503 response moved inside the same gen. This is correctness-load-bearing — the previous shape implicitly assumed a runtime was already set up at module-load time, which is exactly what the façade hid.
4. **`middleware/workspace-routing.ts`** — provides `Workspace.Service` (via its `defaultLayer`) so all downstream handlers in the workspace-scoped route group can `yield* Workspace.Service` without per-handler runtime construction. (~32 added / 25 removed.)
5. **`handlers/sync.ts`, `routes/instance/sync.ts`, `routes/control/workspace.ts`, `server/workspace.ts`** — call-site migrations from `Workspace.create(...)` / `Workspace.list(...)` / `Workspace.get(...)` to `yield* workspace.create(...)` etc., with the service obtained at the top of each handler group.
6. **Tests (~5 files, ~321 added / ~303 removed)** — `httpapi-workspace.test.ts` rewritten end-to-end (211/-227) to use service layers and effectful setup; `httpapi-session.test.ts`, `httpapi-instance-context.test.ts`, `httpapi-workspace-routing.test.ts`, `plugin/workspace-adaptor.test.ts`, `control-plane/workspace.test.ts` all migrated to the same shape. Hono-boundary coverage is explicitly preserved.

## Quality / risk observation
This is the right architectural cut and the diff is honest: the +421/-405 delta is dominated by the test rewrite (211/227) — production code is approximately net-zero in line count but a large net win in clarity, because every consumer now goes through `Workspace.Service` exactly once, in a way the type system can verify. The `proxy.ts` move of the `isSyncing` check into the generator is the most consequential behavioural change — the old `if (!Workspace.isSyncing(workspaceID)) return Effect.succeed(...)` ran *eagerly* at function call time, before any Effect evaluation, which is a subtle category-error in Effect code and was only "working" because the façade had a long-lived runtime backing it. The new shape is correct under all execution orders. The `fence.waitEffect` / `fence.wait` split is a good template for the rest of the codebase: keep the legacy promise-shaped boundary for Hono callers, expose the Effect-shaped primitive for everything else, and have the legacy shim be a one-line `AppRuntime.runPromise(effectVersion(...))`. One nit: `waitEffect` is exported as a named function but not documented as the preferred entry point — a one-line JSDoc on `wait` saying "prefer `waitEffect` from new Effect call sites" would help future contributors avoid re-creating the façade pattern. Verification list runs the four most relevant test groups with an explicit `--test-name-pattern` to keep the failure set inspectable. Zero behavioural regressions visible.

## Verdict
`merge-as-is`
