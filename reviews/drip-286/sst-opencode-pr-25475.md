---
repo: sst/opencode
pr: 25475
head_sha: d7f95cdfdfc760db1e3b53e31e14ee0cf72de3b2
title: "fix(instance): run bootstrap from instance store"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: sst/opencode#25475 — `fix(instance): run bootstrap from instance store`

**Head SHA:** `d7f95cdfdfc760db1e3b53e31e14ee0cf72de3b2`
**Stat:** +216 / −111 across multiple files. Fixes #25450.

## What it changes

The PR moves bootstrap responsibility from `cli/bootstrap.ts` callers
into `InstanceStore.boot`, so every load/reload path runs plugins and
core init *before* exposing the instance. Mechanics:

1. **New module split**: `packages/opencode/src/project/bootstrap-service.ts`
   now holds just the `Service` tag (`@opencode/InstanceBootstrap`); the
   implementation graph stays in `project/bootstrap.ts` and re-exports
   the tag. This breaks the circular import between
   `instance-store` → `bootstrap` → `Config` → `Instance` →
   `instance-store` by letting `instance-store.ts` depend on only the
   lightweight tag file.
2. **New `InstanceRuntime` module** (`project/instance-runtime.ts`)
   replaces the public surface previously exposed by `InstanceStore`
   for `disposeInstance` / `disposeAllInstances` callers
   (`cli/bootstrap.ts`, `cli/cmd/tui/worker.ts`, `config/config.ts`).
3. **`cli/bootstrap.ts`** drops the `init: await getBootstrapRunEffect()`
   parameter from `Instance.provide({...})` — bootstrap now runs inside
   the store on first load instead of being injected per `provide`
   call.
4. **`config/config.ts`** routes through a lazy
   `loadInstanceRuntime()` helper (line 49–51) to avoid pulling the
   runtime graph into config at module init time.
5. **`effect/app-runtime.ts`** drops the `getBootstrapRunEffect`
   memoized promise factory entirely; `AppLayer` now provides
   `InstanceRuntime.layer` instead of `InstanceBootstrap.defaultLayer +
   InstanceStore.defaultLayer`.
6. New regression tests
   (`test/project/instance-bootstrap-regression.test.ts`,
   `test/agent/plugin-agent-regression.test.ts`) lock in the
   "bootstrap runs before instance is exposed" invariant and the
   plugin-registered-agent path that #25450 broke.

## Assessment

- The diagnosis matches #25450 and the symptom that #25466 also fixed
  partially: paths that resolved an instance/agent before the explicit
  `bootstrap` call observed config without plugin mutations. Pushing
  bootstrap *into* the store is the structurally correct fix —
  `#25466` was the local patch; this is the systemic one.
- The tag/impl split via `bootstrap-service.ts` is the right pattern
  to break the import cycle. The comment block at `project/bootstrap.ts`
  around the new `Service` re-export explains exactly why
  (`InstanceStore imports only the lightweight tag from
  bootstrap-service.ts, so it can depend on bootstrap without importing
  this implementation graph`). Important — without that comment, a
  future tidy-up will collapse the two files and reintroduce the
  cycle.
- Lazy `loadInstanceRuntime()` in `config/config.ts:49` is a slight code
  smell (dynamic import to dodge a graph cycle) but is well-isolated
  and only used in two call sites (`disposeInstance` after a failed
  config update, and `invalidate`). Acceptable trade-off.
- Test coverage looks proportionate for a refactor of this scope: a
  boundary-invariant test plus a plugin-agent regression test
  covers both the immediate bug (#25450) and the mechanism that caused
  it.

## Nits

- (Non-blocking) Once #25466 (`fix(agent): initialize plugins before
  resolving config`) lands as a peer fix, that 4-line `yield* plugin.init()`
  in `agent/agent.ts` becomes dead weight — bootstrap-via-store
  guarantees plugins are initialized before any agent layer can
  resolve. Worth a follow-up to remove it (and its comment) so the
  next person reading `agent.ts` doesn't think there's still a
  bootstrap ordering hazard.
- (Non-blocking) `loadInstanceRuntime()` returns
  `Promise<typeof InstanceRuntime>`. `invalidate`'s usage at
  `config/config.ts:752` (`loadInstanceRuntime().then(runtime =>
  runtime.disposeAllInstances())`) drops back into raw promise chaining
  inside an Effect.fn — fine, but inconsistent with the
  `Effect.promise(async () => ...)` pattern used at line 748. Tiny
  cleanup.
- (Non-blocking) `bootstrap-service.ts` re-exports `* as
  InstanceBootstrap from "./bootstrap-service"` — works but creates a
  naming circular reference that some bundlers/IDEs render awkwardly.
  Could just `export { Service }` and let `bootstrap.ts` own the
  namespace re-export.

## Verdict

**`merge-after-nits`** — correct systemic fix for #25450, well-bounded
test coverage, deliberate import-graph design. Ship after the cleanup
nits (or as follow-ups).
