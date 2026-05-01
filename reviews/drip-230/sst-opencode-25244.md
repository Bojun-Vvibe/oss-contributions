# sst/opencode #25244 — fix(app): avoid preview child MCP bootstraps

- **PR**: https://github.com/sst/opencode/pull/25244
- **Head SHA**: `5d5e07eb4212dfd7aa62406206e1c3d642390990`
- **Files reviewed**: `packages/app/src/context/global-sync/child-store.ts`, `packages/app/src/context/global-sync/child-store.test.ts`
- **Date**: 2026-05-01 (drip-230)

## Context

`createChildStoreManager` exposes both a `child(dir, options)` and `peek(dir, options)`
entry point that take a `bootstrap?: boolean` flag. The intent is that when the UI is
hovering / sidebar-previewing a project, the child store is constructed with
`bootstrap: false` so it doesn't kick off directory-scoped backend work. The bug is that
status queries (`path.get`, `mcp.status`, `lsp.status`, `provider.list`) were registered
unconditionally inside `ensureChild`'s `useQueries(() => ({ queries: [...] }))` block at
`child-store.ts:181-185`, so even a preview child immediately fired four GETs at the
backend that themselves bootstrap the directory state on the server side — the `bootstrap:
false` flag was effectively a no-op for the load-bearing side effects.

## Diff

`child-store.ts:37` adds a per-key gate store:

```diff
+ const [bootstrapEnabled, setBootstrapEnabled] = createStore<Record<string, boolean>>({})
```

`child-store.ts:181-187` swaps each query's `sdk` arg for `bootstrapEnabled[key] ? sdk : undefined`:

```diff
- loadPathQuery(key, sdk),
- loadMcpQuery(key, sdk),
- loadLspQuery(key, sdk),
- loadProvidersQuery(key, sdk),
+ loadPathQuery(key, bootstrapEnabled[key] ? sdk : undefined),
+ loadMcpQuery(key, bootstrapEnabled[key] ? sdk : undefined),
+ loadLspQuery(key, bootstrapEnabled[key] ? sdk : undefined),
+ loadProvidersQuery(key, bootstrapEnabled[key] ? sdk : undefined),
```

`child-store.ts:267-282`: both `child()` and `peek()` set `bootstrapEnabled[key] = true`
*before* `ensureChild` runs whenever `shouldBootstrap` is true. The eviction path at
`:108-114` also clears the flag via `produce(draft => delete draft[key])` so a
re-creation doesn't inherit a stale "bootstrapped" bit.

## Observations

1. **Right gate at the right boundary.** The four `loadXQuery` factories already accept
   `undefined` as the sdk arg and return `skipToken` (the test mock at `child-store.test.ts:7`
   even hardcodes that contract). Gating at the query-args site rather than at the
   `useQueries` factory means the reactive shape stays identical (still four query slots,
   just with `skipToken` queryFn) — no Solid lifecycle surprises when a preview child later
   transitions to bootstrapped because the same query slots become live in place.

2. **Promotion path (preview → bootstrapped) works.** If `child(dir)` is called first with
   `{ bootstrap: false }` and later with `{ bootstrap: true }`, the second call hits
   `setBootstrapEnabled(key, true)` at `:270`, which by Solid store reactivity flips the
   four query gates from `skipToken` to `sdk` *without* recreating the child store. The
   test at `:69-130` covers the negative direction (no bootstrap → no calls); a positive
   "promotion fires the queries" test arm would round it out but isn't load-bearing for
   correctness.

3. **Eviction cleanup is the subtle right move.** Without the `setBootstrapEnabled(produce(...))`
   delete at `:111-115`, evicting then recreating a directory key would leave
   `bootstrapEnabled[key] === true` from the prior incarnation, so a subsequent
   `child(dir, { bootstrap: false })` would silently re-fire all four queries. The PR
   correctly clears all four lookup tables (`metaCache`, `iconCache`, `lifecycle`,
   `bootstrapEnabled`) at the same eviction site, matching the existing pattern.

4. **Test mock module replacement is correct.** The new `mock.module("@tanstack/solid-query", ...)`
   at `child-store.test.ts:5-26` provides a `useQueries` impl that actually invokes
   `query.queryFn` only when it isn't `skipToken`, which is what makes the assertion
   `expect(calls).toEqual([])` at `:128` meaningful — without that mock the production
   `useQueries` would never call the queryFn at all under jsdom and the test would pass
   trivially.

## Nits

- The dynamic `await import("./child-store")` at `:18` is necessary because the module-level
  mocks must register before the module loads; a top-level comment explaining "the import
  must happen after mock.module so the production tanstack import resolves to the mock" would
  save the next reader the round trip.
- `bootstrapEnabled[key] ? sdk : undefined` is repeated four times; a local
  `const sdkIfBootstrapped = bootstrapEnabled[key] ? sdk : undefined` would dedupe and
  avoid the read-four-times-in-one-tick reactivity reading pattern.

## Verdict

merge-after-nits
