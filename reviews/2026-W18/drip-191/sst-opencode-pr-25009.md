# sst/opencode #25009 — feat(project): add DELETE /project/:id endpoint

- **URL:** https://github.com/sst/opencode/pull/25009
- **Head SHA:** `dd0fbd0b36d5c00dd51f88d7243d0c4dade3a124`
- **Files:** `packages/opencode/src/project/project.ts` (+19/-1), `packages/opencode/src/server/routes/instance/httpapi/groups/project.ts` (+15/-0), `packages/opencode/src/server/routes/instance/httpapi/handlers/project.ts` (+12/-1), `packages/opencode/src/server/routes/instance/project.ts` (+32/-0), `packages/opencode/test/project/project.test.ts` (+31/-0), `packages/opencode/test/server/httpapi-instance.test.ts` (+20/-0)
- **Verdict:** `merge-after-nits`

## What changed

Adds a `DELETE /project/:projectID` endpoint on both the legacy Hono surface (`server/routes/instance/project.ts:121-146`) and the new HttpApi surface (`server/routes/instance/httpapi/groups/project.ts:60-74` + `handlers/project.ts:44-48` + `:99-103`). The service-layer `remove` is at `project.ts:469-484`:

```typescript
const remove = Effect.fn("Project.remove")(function* (id: ProjectID) {
  const existing = yield* db((d) => d.select().from(ProjectTable).where(eq(ProjectTable.id, id)).get())
  if (!existing) throw new NotFoundError({ message: `Project not found: ${id}` })
  const sessionsRow = yield* db((d) =>
    d.select({ count: sql<number>`count(*)` })
      .from(SessionTable)
      .where(eq(SessionTable.project_id, id))
      .get(),
  )
  const sessionsRemoved = sessionsRow?.count ?? 0
  yield* db((d) => d.delete(ProjectTable).where(eq(ProjectTable.id, id)).run())
  return { deleted: true, sessionsRemoved }
})
```

The Hono route at `:127-133` documents the contract: "Cascading foreign keys handle all child data." Tests cover the round-trip at `project.test.ts:600-633` (success, NotFound, list-removal) and the HttpApi bridge at `httpapi-instance.test.ts:143-163`.

## Why it's worth landing

- Real ergonomic gap. There was no way to delete a project from the API surface; users had to manipulate the SQLite file by hand. The cascade-delete at the SQL level (asserted by sessions count in the response) gives operators the visibility to verify the destructive op did what they expected.
- Symmetric API surface. Both Hono and HttpApi paths get the new operation in the same PR — no surface drift.
- Response shape `{ deleted: boolean, sessionsRemoved: number }` is honest: the boolean is the "did we touch the row" signal, and `sessionsRemoved` is the cascade-count operators care about for "did I just nuke a thousand sessions" sanity-check.

## Nits / blockers worth fixing

1. **`sessionsRemoved` is a count snapshot, not a delete count.** The `sessionsRow?.count` at `:476` is read *before* the `db.delete(ProjectTable)` at `:480`. Because the delete runs asynchronously through the cascade, there's an implicit assumption that "count of sessions matching `project_id = id`" equals "sessions that will be cascaded." That's true *only* if the FK is `ON DELETE CASCADE` and no other writers can insert sessions between the count read and the parent delete. A SQLite transaction wrapping both operations would close that window. As written, the response field can lie under load.

2. **`throw new NotFoundError(...)` inside `Effect.fn` body at `:472`.** Effect convention is `yield* Effect.fail(new NotFoundError(...))` — bare `throw` works because Effect catches synchronous exceptions, but it skips the typed-error track and shows up as a defect (`Cause.Die`) rather than a tagged failure. Most callers route through the Hono `jsonRequest` or HttpApi `HttpApiBuilder.handle`, which in this codebase do tend to convert defects to 500 — meaning the HttpApi success schema `{deleted: Boolean, sessionsRemoved: Number}` at `groups/project.ts:62-65` doesn't declare the 404 case at all. Compare to peer endpoints that declare `errors(400, 404)` like the Hono route at `project.ts:138`. The HttpApi group needs a `Schema.Union` or explicit `error: NotFoundErrorSchema` to round-trip the 404 honestly.

3. **No HttpApi negative-path test.** `httpapi-instance.test.ts:143-163` covers only the happy path; there's no test for "DELETE non-existent ID returns 404" on the HttpApi surface specifically. The legacy Hono test at `project.test.ts:613-617` does cover the NotFound path, but only at the service layer.

4. **Trailing newline at `groups/project.ts:92`** — empty line at end of file inside the diff slice. Just a cleanup.

5. **Test at `project.test.ts:606`** asserts `expect(Project.get(project.id)).toBeUndefined()` synchronously, but `Project.get` (in this codebase) is typically a yield-able effect. Verify this isn't accidentally returning a `Promise` that always passes the `toBeUndefined()` check.

## Risk

Moderate. The destructive operation is bounded to a single row + cascade, but the snapshot-then-delete pattern at `:475-480` means the response field can be stale under a concurrent insert. For an operator-facing destructive endpoint that's livable but worth fixing before users start automating against it. The 404-shape gap on the HttpApi surface (nit #2) is the kind of thing that becomes load-bearing the moment a SDK consumer relies on the typed error.
