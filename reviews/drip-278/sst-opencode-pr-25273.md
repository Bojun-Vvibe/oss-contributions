# Review — sst/opencode#25273

- PR: https://github.com/sst/opencode/pull/25273
- Title: refactor(desktop-electron): improve main process architecture
- Head SHA: `46cd9c09cd9eed785d8b34f535a77bb4c2b8351a`
- Size: +410 / −303 across 7 files
- Verdict: **needs-discussion**

## Summary

Rewrites the Electron main process from an imperative
`EventEmitter` + `defer`-based control flow to an Effect-TS
fiber-based composition. Replaces the ad hoc `initEmitter`,
`loadingComplete`, `serverReady`, `pendingDeepLinks: string[]`
mutables with `PubSub`, `Deferred`, `SubscriptionRef`, `Queue`, and
`Stream` primitives from the `effect` package, and switches
`spawnLocalServer` to an Effect-returning `spawnLocalServerEffect`.

## Evidence

- `packages/desktop-electron/src/main/index.ts:8` — adds
  `import { Data, Deferred, Effect, Fiber, Option, PubSub, Queue,
  Ref, Stream, SubscriptionRef } from "effect"`. This is a load-bearing
  new runtime dependency on the entire `effect` ecosystem in the main
  process bundle.
- Same file, lines ~36-49 — removes the module-level `initEmitter`,
  `initStep`, `mainWindow`, `server`, `loadingComplete`,
  `pendingDeepLinks`, `serverReady` mutables. These are presumably
  re-expressed as `Ref` / `SubscriptionRef` / `Deferred` /
  `PubSub` / `Queue` further down (the diff is mostly a wholesale
  replacement of the bottom half of the file).
- `packages/desktop-electron/src/main/server.ts` — old
  `spawnLocalServer` is replaced with `spawnLocalServerEffect`, called
  from the new Effect program in `index.ts`.
- `packages/desktop-electron/src/preload/types.ts` — `InitStep`,
  `ServerReadyData`, `SqliteMigrationProgress`, `WslConfig` are now
  imported as values rather than `import type`, suggesting the new
  code uses them as runtime tags (likely with `Data.TaggedEnum` for
  exhaustive matching).

## Concerns / why "needs-discussion"

- **Bundle size & cold-start.** `effect` plus its peer modules adds a
  non-trivial chunk to the main process bundle. Electron main process
  startup is on the user-visible critical path (window paint waits on
  it). No before/after numbers for `app.ready` latency or main bundle
  size are in the PR. Please attach those.
- **Single-file mega-refactor.** 7 files, ~700 lines churn, one
  commit, no incremental boundary. Reviewing semantic equivalence of
  the old `defer<ServerReadyData>()` → new `Deferred.make<…>()` plus
  fiber lifecycle requires reading both states end to end. Splitting
  into (1) introduce Effect runtime + adapters, (2) port deep-link
  queue, (3) port server-ready handshake, (4) port loading window
  would dramatically improve reviewability.
- **Error-path semantics.** The old code's `app.quit()` on
  `requestSingleInstanceLock()` failing, and the implicit "uncaught
  promise rejection crashes main" behavior, both change shape under
  Effect (errors travel as `Cause` inside fibers). The PR needs an
  explicit story for: who interrupts the root fiber on `app.quit`,
  what happens to in-flight `spawnLocalServerEffect` on shutdown, and
  whether `Fiber.interrupt` is wired into the relevant `app` lifecycle
  events.
- **Interop with `electron-updater`, `dialog`, `BrowserWindow`.**
  These remain promise-/callback-based; the PR should clarify whether
  they're wrapped in `Effect.async` / `Effect.tryPromise` consistently
  or only opportunistically.

## Recommendation

Hold for: (a) bundle-size + startup-latency numbers, (b) split into
≥3 commits with intermediate green CI, (c) an explicit shutdown /
fiber-interruption story documented in the PR body. The architectural
direction is reasonable but the PR as-shaped is hard to land safely.
