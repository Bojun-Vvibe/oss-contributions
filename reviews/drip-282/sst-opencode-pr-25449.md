# sst/opencode PR #25449 — fix(instance): restore InstanceBootstrap init parameter for non-Effec…

- Head SHA: `5e417a8133f40c65f964e3bb066429900ebb3bd4`
- Size: +27 / -6, 7 files

## Specific refs

- `packages/opencode/src/effect/app-runtime.ts:133-144` — new lazy `getBootstrapRunEffect()` resolves `InstanceBootstrap.Service.run` once via `AppRuntime.runPromise`, caches the resulting `Promise<Effect<void>>`. Cache lives at module scope so subsequent calls are zero-cost.
- `packages/opencode/src/effect/app-runtime.ts:95` — adds `InstanceBootstrap.defaultLayer` to `AppLayer` so the service is actually resolvable.
- `packages/opencode/src/agent/agent.ts:83` — defense-in-depth `yield* plugin.init()` placed inside the `Agent.state` builder where `InstanceRef` ctx is available; `plugin.init()` is documented idempotent.
- `packages/opencode/src/server/routes/instance/middleware.ts:26`, `cli/bootstrap.ts:8`, `cli/cmd/tui/worker.ts:80`, `server/workspace.ts:97-103`, `server/routes/instance/project.ts:85` — all five legacy callsites wired to pass `init: await getBootstrapRunEffect()`.

## Assessment

Real regression, narrow surgical fix, root cause clearly explained. Concern: `bootstrapRun` is a single module-level promise — if `AppRuntime.runPromise` ever rejects, the cached rejection is sticky for process lifetime (no retry path). For a bootstrap effect that's probably acceptable (a failed bootstrap is fatal anyway), but worth a one-line comment. The defense-in-depth `plugin.init()` inside the state builder is correct placement — putting it at layer init time would not have `InstanceRef` available.

Pre-existing `spinner.ts` typecheck error mentioned by the author is unrelated.

verdict: merge-after-nits