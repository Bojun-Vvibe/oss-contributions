# Review: sst/opencode #25434 — feat(models): effectify ModelsDev as Service

- **PR**: https://github.com/anomalyco/opencode/pull/25434
- **Head SHA**: `fdaf026eba9f3245ef40081df55248ab5ee12d13`
- **Diff size**: ~330 lines, primary file `packages/opencode/src/provider/models.ts`

## What it changes

Refactors `ModelsDev` from a module-level lazy + Promise-returning `get()`/`refresh()` into
an Effect `Context.Service` with a real `Layer`. Concretely:

- Removes the module-level `lazy(...)` cache, the `setInterval(..., 60*60*1000).unref()`
  background refresher, and the bare `fetchApi()` async function.
- Defines `Service` (`models.ts:84`) with `get`/`refresh` operations.
- New `layer` at `models.ts:90-240` builds the service from `AppFileSystem.Service` and
  `HttpClient.HttpClient`, uses `Effect.cachedInvalidateWithTTL(populate, Duration.infinity)`
  for single-flight caching, `Flock.effect(lockKey)` for cross-process locking, and
  `Effect.forkScoped` + `Schedule.fixed("60 minutes")` for the background refresh.
- Keeps a Promise-style compatibility shim (`models.ts:251-252`) backed by
  `makeRuntime(Service, defaultLayer)` so legacy Hono routes / CLI handlers still work.
- Wires `ModelsDev.defaultLayer` into `AppLayer` (`effect/app-runtime.ts:70`),
  `Provider.layer` (`provider/provider.ts:264`, providing it via `Layer.provide` at 1726),
  the HTTP API server (`server/routes/instance/httpapi/server.ts:159`), and switches
  `Effect.promise(() => ModelsDev.get())` call sites to `ModelsDev.Service.use((s) => s.get())`
  at `provider.ts:280`, `httpapi/handlers/provider.ts:300`, `instance/provider.ts:333`.
- `cli/cmd/models.ts:11` switches the `--refresh` handler away from
  `Effect.promise(() => ModelsDev.refresh(true))` to `ModelsDev.Service.use((s) => s.refresh(true))`.

## Assessment

This is the right direction. The previous implementation had three latent issues that this
PR cleans up:

1. **Module-load side effects**: `setInterval(..., 60*60*1000).unref()` fired at import
   time, which in test environments meant the timer escaped past `vi.useFakeTimers()`
   setup and bled state across suites. Moving it inside `Effect.forkScoped` (line 233)
   ties the timer's lifecycle to the runtime scope.
2. **TTL semantics**: the old `fresh()` checked mtime against `5 * 60 * 1000`, then
   `refresh()` re-checked inside the lock — fine, but the TTL was a magic number. New
   code uses `Duration.minutes(5)` (line 107) which is more legible.
3. **HTTP retries**: the old `fetchApi` did a bare `fetch(...)` with `AbortSignal.timeout(10000)`
   and no retry on transient network blips. The new path runs through
   `withTransientReadRetry(yield* HttpClient.HttpClient)` (line 93) and
   `HttpClient.filterStatusOk`, picking up whatever retry policy the rest of the codebase
   uses for transient HTTP reads.

Concerns to raise before merge:

- **`Effect.cachedInvalidateWithTTL(populate, Duration.infinity)` at line 212**: with
  `Duration.infinity`, the cache never expires by TTL — only `invalidate` clears it. The
  `refresh` function (line 216) explicitly invalidates after writing the file. That's
  correct. But the comment on line 211 says "single-flight cache: concurrent first-call
  dedupes via the cached effect," which is the right reason to use cached, just not the
  same as a TTL-based cache. Worth disambiguating: this is a "compute once, hold forever
  unless explicitly invalidated" cache, not a TTL cache. The `WithTTL` in the API name is
  misleading at the call site here.
- **Background refresh at line 234**: `refresh().pipe(Effect.andThen(refresh().pipe(Effect.repeat(...))), Effect.ignore)`.
  The eager `refresh()` runs once, *then* a second `refresh()` repeats every 60 min. That
  means the first scheduled refresh fires immediately after the eager one (zero delay
  before the schedule's first tick). Probably not what you want — `Schedule.fixed` starts
  with the first action immediately. Consider `Schedule.spaced("60 minutes")` or
  `Effect.delay("60 minutes")` before the repeat.
- **Promise-compat shim at line 250-252**: comment claims "makeRuntime uses the shared
  memoMap so this runtime's Service instance is the same one AppRuntime sees." If that's
  true, great — but it's a load-bearing claim. A test that calls `ModelsDev.get()` (the
  promise shim) and then via `Service.use((s) => s.get())` from a runtime that has a
  pre-warmed cache, asserting they hit the same cached value, would lock this in.
- **`loadSnapshot` at line 170-174**: `Effect.tryPromise({ try: ..., catch: () => undefined })`
  followed by `.pipe(Effect.catch(() => Effect.succeed(undefined)))`. The double-catch is
  redundant — `tryPromise.catch` already converts thrown errors. Drop one of them.

## Verdict

`merge-after-nits` — the refactor is well-conceived and the call-site migration is
complete. Address the misleading TTL comment, the immediate-fire schedule, and ideally
add the runtime-sharing test. No blocking issues; the API surface stays compatible via
the Promise shim, and the Effect-native call sites get proper dependency injection.
