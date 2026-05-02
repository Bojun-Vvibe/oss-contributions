# sst/opencode #25389 — fix(opencode): File watch stop working in dev

- **Head SHA:** `f3c5d07b67c18ce3d9508f208ab300dcb63bad5b`
- **Files:** `packages/opencode/src/cli/bootstrap.ts`, `packages/opencode/src/cli/cmd/tui/worker.ts`, `packages/opencode/src/effect/app-runtime.ts`, `packages/opencode/src/server/routes/instance/middleware.ts`, `packages/opencode/src/server/workspace.ts`, `packages/opencode/test/file/watcher.test.ts` (+39/-0)
- **Verdict:** `merge-after-nits`

## Rationale

Root cause read is correct: `Instance.provide` was being called without an `init` callback in dev paths (CLI bootstrap, TUI worker upgrade probe, server instance middleware, workspace route), so the `InstanceBootstrap` Effect service that wires up the file watcher never ran. The dev binary worked the first time a server route fired the middleware but any earlier code path (CLI subcommand, worker `checkUpgrade`, direct workspace use) skipped bootstrap and the watcher silently never started. Adding `init: () => AppRuntime.runPromise(InstanceBootstrap.Service.use((bootstrap) => bootstrap.run))` at four call sites and registering `InstanceBootstrap.defaultLayer` into `AppLayer` (`app-runtime.ts:94`) is the minimum surgical fix.

Nits: (1) the `init` lambda is identical at four call sites — extract a `defaultInstanceInit` helper in `project/bootstrap.ts` to keep the four sites in lockstep when a future bootstrap step is added; (2) the new `watcher.test.ts` is +39 lines and should explicitly assert that bootstrap was called *before* the first file event is dispatched, not just that events arrive (otherwise the regression this PR fixes can silently come back if init order shifts); (3) confirm the production server path (which already worked) still calls `init` exactly once per `Instance.provide` and not once per route — the middleware lives on every request.

Risk is low: the change adds an init hook that was missing, doesn't alter the watcher itself, and the failure mode is "bootstrap is invoked an extra time" rather than "file events double-fire" because `InstanceBootstrap.Service.use` is idempotent inside the AppRuntime layer.
