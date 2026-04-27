# sst/opencode PR #24484 — bridge sync routes onto HttpApi

- **PR**: https://github.com/sst/opencode/pull/24484
- **Author**: @kitlangton
- **Head SHA**: `91ca8d67e0355ca5ecc414e525dcf3bc3062a448`
- **Size**: +246 / −25
- **Files**: spec checklist + new `httpapi/sync.ts`, edits to `httpapi/server.ts`, `httpapi/mcp.ts`, `httpapi/workspace.ts`, plus `test/server/httpapi-sync.test.ts`

## Summary

Bridges the three workspace sync endpoints — `POST /sync/start`, `POST /sync/replay`, `POST /sync/history` — from the legacy Hono router onto `SyncApi` (a new `HttpApi.make("sync")` group). Net new file: `packages/opencode/src/server/routes/instance/httpapi/sync.ts` (130 lines). Side improvement: the `mcp.add` handler is rewritten from `Schema.decodeUnknownSync` to `Schema.decodeUnknownEffect` with a `mapError -> HttpApiError.BadRequest`, fixing a latent throw-from-handler bug.

## Verdict: `merge-as-is`

Cleanly scoped, mirrors the established pattern from prior stack PRs (`session-routes`, `workspace-routes`), and the spec checklist + status table at `packages/opencode/specs/effect/http-api.md:170-188` is updated to flip `sync` from `later` to `bridged` in the same diff. The `mcp.add` decode-effect change is a real bugfix worth landing on its own merits — `decodeUnknownSync` would have thrown synchronously inside an `Effect.fn`, which would surface as an unhelpful 500 instead of a typed 400.

## Specific references

- `packages/opencode/src/server/routes/instance/httpapi/sync.ts:1-12` — imports pull `Database, asc, and, eq, lte, not, or` from `@/storage` plus `EventTable` from `@/sync/event.sql`. The new file is doing direct SQL composition for `/sync/history`, not delegating to a `Sync.history()` service method. That's a deliberate choice that keeps the bridge thin, but it does mean the SQL query lives in the HTTP layer — fine for now, but if `/sync/history` ever needs to be reused (e.g. by an event-replay CLI), the body should move into `Sync` proper.
- `packages/opencode/src/server/routes/instance/httpapi/server.ts:21,77` — `SyncApi`/`syncHandlers` are added to the imports and registered via `HttpApiBuilder.layer(SyncApi).pipe(Layer.provide(syncHandlers))`, sandwiched between `ProviderApi` and `WorkspaceApi`. Registration order is alphabetic-ish; consistent with neighbors.
- `packages/opencode/src/server/routes/instance/httpapi/mcp.ts:139-145` — the genuine fix:
  ```ts
  // before:
  const payload = Schema.decodeUnknownSync(AddPayload)(ctx.payload)
  // after:
  return yield* Schema.decodeUnknownEffect(StatusMap)(...)
    .pipe(Effect.mapError(() => new HttpApiError.BadRequest({})))
  ```
  This flips a panic-on-bad-shape into a typed 400. Note the `before` removed the `Schema.decodeUnknownSync(AddPayload)` line entirely — the framework decoder now runs from the endpoint's `payload:` annotation, so a manual decode was redundant. Net: less code and better errors. Worth calling out in the PR description (currently empty besides the gh-stack footer).
- `packages/opencode/specs/effect/http-api.md:184` — table row for `sync` flips from `later` / "process/control side effects" to `bridged` / "start/replay/history". Checklist items at `:283-285` and implementation-order item 7 at `:356` also flip in lockstep.

## Nits

1. The PR body is just the gh-stack auto-footer. Even one sentence ("Bridges sync; also fixes a latent decode-throw in mcp.add") would help reviewers and the future bisect target.
2. `httpapi/sync.ts:130` lines is right at the boundary where extracting the SQL into a `Sync.list({ before, limit })` helper starts to pay for itself. Consider doing it in a follow-up before `/sync/replay` grows.

## What I learned

Spotted a real pattern: when migrating off a sync-throwing decoder library to one that returns `Effect`/`Result`, you almost always find latent bugs in the call sites that were "trusted" the input shape. The `mcp.add` fix was incidental to a sync-routes PR — that's the kind of bonus you only get when each migration step is a same-shape mechanical edit done by someone reading the original carefully. Argues for doing these strangler migrations one route group at a time, not en masse.
