# sst/opencode #25853 — chore(desktop): make proxy setup order explicit

- **Head SHA:** `a8db60bac35d4f88208ae26511e704a576edd74b`
- **Base:** `dev`
- **Author:** Hona
- **Size:** +34 / −39 across 2 files (`packages/desktop/src/main/index.ts`, `packages/desktop/src/main/server.ts`)
- **Verdict:** `merge-as-is`

## Summary

Pure refactor follow-up to #25846. Consolidates the two-step "ensure
loopback NO_PROXY → call `http.setGlobalProxyFromEnv()`" dance into a
single `configureProxyEnv()` exported from `server.ts`, and removes the
callback parameter from `spawnLocalServer`. Net behavior is identical;
only the call-site shape changes.

## What's right

- **Order is now an invariant of the function, not the caller.** Before
  this PR, `setupApp` called `ensureLoopbackNoProxy()` then
  `useEnvProxy()` (`index.ts:80-81` pre-PR), and `initialize()` did the
  same two-call sequence again via the `configureEnv` callback before
  spawning the sidecar. Two callers, same ordering rule, easy to drift.
  After: `configureProxyEnv()` does loopback-upsert *then*
  `setGlobalProxyFromEnv()` exactly once per call site
  (`server.ts:80-106`). That ordering matters — `setGlobalProxyFromEnv()`
  reads `NO_PROXY` at invocation time, so loopback must already be in
  the env when it fires, otherwise the embedded server's first internal
  fetch can be routed through the corporate proxy.

- **Callback removal is the right shape.** The pre-PR
  `spawnLocalServer(..., configureEnv?: () => void)` signature
  (`server.ts:33`) made proxy setup an *optional* injected concern; the
  new `spawnLocalServer` calls `configureProxyEnv()` directly at
  `server.ts:36-37`. This makes it impossible for a future caller of
  `spawnLocalServer` to forget the env step and silently break corporate
  proxy users — exactly the failure mode the original callback design
  invited.

- **`setupApp` simplification is congruent.** `index.ts:79` now calls
  `configureProxyEnv()` once before `app.commandLine.appendSwitch(...)`
  — the same ordering as before, just expressed as one call instead of
  two, with the helpers `ensureLoopbackNoProxy` and `useEnvProxy`
  deleted from `index.ts` (lines 136-141 and 289-307 in the pre-PR
  version) so there's no zombie copy that could drift from the
  `server.ts` version.

- **Logger swap is correct.** The deleted `useEnvProxy()` in `index.ts`
  used the file-local `logger` (initialized via `initLogging`); the new
  `configureProxyEnv` in `server.ts` imports `electron-log/main.js`
  directly (`server.ts:3`) because `server.ts` had no logger of its own
  before. Same underlying transport, no log-routing regression.

- **`http` import migration is clean.** The `import * as http from
  "node:http"` moves from `index.ts` (deleted at line 4) to `server.ts`
  (added at line 1) — the type-cast comment about `Electron 41.2 / Node
  24.14.1 / @types/node@24.12.2` is preserved verbatim at
  `server.ts:103-104`, so the next person bumping `@types/node` knows
  why the `(http as any)` cast exists.

## Nits (non-blocking)

- The `configureProxyEnv` JSDoc would help future maintainers — right
  now the function's two-phase ordering invariant (loopback-upsert
  *before* `setGlobalProxyFromEnv()`) is only documented by the comment
  on the type cast, not by the function contract itself.

- The deleted `useEnvProxy` had a `logger.warn("failed to load proxy
  environment", error)`; the new `configureProxyEnv` has the equivalent
  `log.warn("failed to load proxy environment", error)` at
  `server.ts:104`. Would be nice to also log the success path once at
  startup so operators debugging "is the corporate proxy actually
  loaded?" have a positive signal, not just absence-of-warning.

## Risks

None worth blocking on. This is a structural refactor with the
callback/inline duplication collapsed to a single function — runtime
semantics are byte-identical because the same two operations
(`upsert NO_PROXY` and `setGlobalProxyFromEnv()`) run in the same order
at the same two trigger points (`setupApp` and `spawnLocalServer`).

## Verdict reasoning

`merge-as-is`: small, motivated by a real-world drift hazard the prior
callback design created, leaves the codebase in a state where
"forgetting proxy setup" is no longer a possible bug. Tests already pass
(`bun typecheck` per PR body). No behavior change for end users.
