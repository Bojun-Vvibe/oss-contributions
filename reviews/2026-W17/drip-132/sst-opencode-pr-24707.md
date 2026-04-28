# sst/opencode PR #24707 — Add Effect Drizzle SQLite adapter

- **PR**: https://github.com/sst/opencode/pull/24707
- **Author**: @kitlangton
- **Head SHA**: `a03a08abe0d63e3aaa2fc61dcbf75d159140afa3`
- **Size**: +639 / −50 across 14 files (new package `packages/effect-drizzle-sqlite/`)

## Summary

Introduces a new internal package `@opencode-ai/effect-drizzle-sqlite` that monkey-patches Drizzle ORM's SQLite query-builder prototypes so the existing query-builder objects are *also* yieldable as `Effect.Effect<T, EffectDrizzleQueryError>`. Existing call sites that previously did `await db.select()...` can now do `yield* db.select()...` inside `Effect.gen` blocks, with errors automatically wrapped with the SQL text and parameter list. Also adds `db.withTransaction` that takes an Effect, runs it inside a synchronous Drizzle transaction via `runSyncExit`, and propagates failures as a sentinel `TransactionFailure` to roll back. Migrates a handful of existing call sites (`session/todo.ts`, `share/share-next.ts`, `permission/index.ts`) to the new shape.

## Verdict: `merge-after-nits`

The shape is the correct one for an opencode-internal Effect/Drizzle bridge — it preserves the existing fluent Drizzle API verbatim, doesn't fork or wrap query builders, lets call sites incrementally adopt `yield*`, and the error-class carries `query`/`params`/`cause` so failed queries are debuggable instead of opaque. The 237-line test file at `packages/effect-drizzle-sqlite/test/sqlite.test.ts` is dense and exercises the load-bearing cells (select/insert/update/delete, relational queries, nested transactions, defects rolling back, query-error wrapping carrying `query` and `params`). But four real concerns deserve to be addressed before merge.

## Specific references

- `packages/effect-drizzle-sqlite/src/index.ts:231-250` — `patchQueryBuilders` is the module's load-bearing piece. It mutates `SQLitePreparedQuery.prototype`, `SQLiteSelectBase.prototype`, `SQLiteInsertBase.prototype`, etc. via `Object.assign(ctor.prototype, queryEffectProto, ...)` and is **process-global**. Any other consumer of `drizzle-orm/bun-sqlite` in the same process — including the test at `:393-407` that explicitly creates a `drizzleBun({ client: sqlite })` *without* going through `make()` — will also see these prototypes mutated. The test passes because Drizzle's `.run()` / `.all()` paths still work, but the assertion `await normal.select().from(users)` at `:404` is now actually exercising the new `Effect`-based `[Symbol.iterator]` path (the patched class extends `Effect.Effect`), not Drizzle's native promise-thenable path. This is the kind of monkey-patch-the-world dependency that breaks downstream packages in surprising ways and is hard to debug.

- `packages/effect-drizzle-sqlite/src/index.ts:217-219` — `[EffectEvaluate]` returns `Effect.die("Drizzle SQLite query is missing asEffect()")` when `asEffect` is missing. This is correct fail-loud behavior, but the `patched` flag at `:232` means a query builder constructed *before* the first `make()` call will lack `asEffect` and dies. In practice `make()` is called at module-init for every consumer, so this is mostly theoretical, but a test asserting "first construction inside `make()` works" would pin it.

- `packages/effect-drizzle-sqlite/src/index.ts:264-288` — `withTransaction` uses `Effect.runSyncExit(Effect.provideContext(effect, context))` inside the synchronous Drizzle transaction callback. This **forbids any async work inside the transaction body** — any `Effect.promise`, `Effect.tryPromise`, or `Effect.sleep` will throw `AsyncFiberException` at runtime. The `EffectSQLiteDatabase` type at `:106-115` doesn't communicate this restriction. At minimum, document it on the `withTransaction` field; ideally encode it in the type with a `Sync` constraint marker.

- `packages/effect-drizzle-sqlite/src/index.ts:255-257` — `txStack` is a per-`db`-instance closure-captured array. The `Proxy` at `:290-299` returns `current()` (the innermost transaction) for every property access. This means a concurrent caller running an Effect *outside* a transaction while another fiber is inside `withTransaction` will see the inner transaction's view of the database — a real race even though Drizzle's transaction is itself synchronous, because `Effect.context()` at `:268` yields to the runtime before `runTransaction(current())(...)` executes. Worth a focused test asserting "concurrent reads outside a transaction don't see inside-transaction writes."

## Migration nits

- `packages/opencode/src/session/todo.ts` — 24 lines deleted, 24 added. Diff is byte-for-byte the `await` → `yield*` translation. Good — no semantic drift.
- `packages/opencode/test/permission/next.test.ts:1-7` (added) — only 6-line change but doesn't pin the new error-wrapping shape. If the test repo cares about catching `EffectDrizzleQueryError` specifically (e.g. matching on `error._tag === "EffectDrizzleQueryError"`), that's worth a regression test in the consuming package, not just the adapter package.
- The new package has zero `AGENTS.md` / `README.md` / `CLAUDE.md` documentation explaining the prototype-patching surprise. The `packages/llm/AGENTS.md` in the sibling stacked PR #24712 sets the right precedent for "this is what surprised you would be" docs.

## Suggested follow-ups

1. Either gate the prototype patch behind an opt-in (e.g. `make({ patchPrototypes: true })`) defaulting to true so consumers can detect they've been patched, or add a one-paragraph `AGENTS.md` warning that *importing* this package's `make` is a global side effect.
2. Add a focused test for "Effect inside `withTransaction` that does `Effect.tryPromise` should fail loudly with a typed error, not panic with `AsyncFiberException`."
3. Consider naming the error class `EffectDrizzleSqliteQueryError` so future Postgres/MySQL adapters in the same family don't collide on the bare `EffectDrizzleQueryError` name.
