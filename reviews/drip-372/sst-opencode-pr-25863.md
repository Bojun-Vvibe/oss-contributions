# sst/opencode#25863 — feat(opencode): add session backup API

- **URL:** https://github.com/sst/opencode/pull/25863
- **Head SHA:** `773a3b7ed9e972d7d204cc23c03f3c037c43261f`
- **Closes:** #20117
- **Files touched:** 11 (+836 / −48)
  - `packages/opencode/src/backup/index.ts` (+132 / −0, new)
  - `packages/opencode/src/effect/app-runtime.ts` (+2 / −0)
  - `packages/opencode/src/server/routes/instance/backup.ts` (+101 / −0, new)
  - `packages/opencode/src/server/routes/instance/httpapi/api.ts` (+2 / −0)
  - `packages/opencode/src/server/routes/instance/httpapi/groups/backup.ts` (+76 / −0, new)
  - `packages/opencode/src/server/routes/instance/httpapi/handlers/backup.ts` (+39 / −0, new)
  - `packages/opencode/src/server/routes/instance/httpapi/server.ts` (+4 / −0)
  - `packages/opencode/src/server/routes/instance/index.ts` (+6 / −1)
  - `packages/opencode/test/server/httpapi-backup.test.ts` (+202 / −0, new)
  - `packages/sdk/js/src/v2/gen/sdk.gen.ts` (+126 / −0, generated)
  - `packages/sdk/js/src/v2/gen/types.gen.ts` (+146 / −47, generated)

## Summary

Adds a session-level backup API exposed both through Hono routes (`POST /backup/{list,export,import}`) and the parallel HttpApi surface. Export returns the full session payload (info + messages-with-parts + todos). Import always allocates a new `SessionID`, `MessageID`, and `PartID`, so importing never overwrites an existing session by accident. Also addresses the todo round-trip gap from #20117 by making todos part of the exported/imported payload.

## Line-level observations

- `packages/opencode/src/backup/index.ts:14-19` — `Payload` schema is `Schema.Struct({ info, messages: Schema.Array(WithParts), todos: Schema.Array(Todo.Info) })`. Reuses existing schemas verbatim, so backups stay shape-compatible with whatever lives in the DB.
- `:32-38` and `:40-48` — `messageValue()` and `partValue()` strip the inbound `id`/`sessionID`/`messageID` via destructure-and-discard before re-keying with freshly generated IDs from `MessageID.ascending()` / `PartID.ascending()`. This is the core "no accidental overwrite" guarantee. Clean.
- `:50-79` — `apply()` runs the four inserts (`SessionTable`, `MessageTable`, `PartTable`, `TodoTable`) inside `Database.transaction(...)`, so a partial import can never half-restore a session. Also explicitly clears `workspaceID` (correct — workspace identity is environment-local) but preserves `projectID` from the request, not the payload (also correct — backup may have come from a different project).
- `:82-117` — `layer` wires `list` / `exportSession` / `importSession`. `exportSession` enforces `info.projectID !== projectID → NotFoundError` at `:110`, which is the correct privacy boundary (caller can't read a session from another project even by guessing the SessionID).
- `:120-122` — `importSession` does `structuredClone(decodePayload(payload))` before passing to `apply()`. The `structuredClone` is defensive against the caller mutating the payload after the call returns; `decodePayload` is `Schema.decodeUnknownSync(Payload)` so any malformed input throws synchronously inside the Effect.
- `packages/opencode/src/server/routes/instance/backup.ts:56-66` — `/export` Hono handler validates `{ sessionID }` and returns the full payload as JSON. No streaming, so a session with hundreds of MB of message parts will buffer entirely server-side then again client-side. Acceptable for v1 but worth noting for follow-up.
- `:78-95` — `/import` handler `structuredClone`s the body again before passing into the service (defensive double-clone — the inner `apply` also clones via `decodePayload`'s `decodeUnknownSync`). Slight wastefulness, but bounded by request body size.
- `packages/opencode/src/server/routes/instance/httpapi/groups/backup.ts` and `handlers/backup.ts` mirror the Hono routes through the HttpApi surface for SDK generation. The two surfaces deliberately stay in sync.
- `packages/sdk/js/src/v2/gen/{sdk,types}.gen.ts` — regenerated. Diff is mechanical (new `Backup*` types and `backupList` / `backupExport` / `backupImport` client methods).
- `packages/opencode/test/server/httpapi-backup.test.ts:1-202` — covers list-returns-sessions, export-returns-payload, export-rejects-cross-project (404), import-creates-new-session, round-trip preserves title and message count, and todos round-trip correctly. Solid coverage.

## Concerns

- No size limit on `/import` request body. A pathological client could POST a 10GB JSON document and OOM the proxy/server. Consider adding a `bodyLimit` middleware on this route group, or at least documenting the practical ceiling.
- `Backup.Payload.zod` is exposed in the OpenAPI schema as the request body for `/import`. The schema accepts arbitrary `MessageV2.WithParts` and `Todo.Info` shapes — fine for trust-the-caller deployments (single-user CLI) but worth flagging if this server is ever exposed multi-tenant.
- `/export` returns the entire payload synchronously. For a large session this is a single multi-MB JSON response. A streaming variant (NDJSON or chunked) would be a reasonable follow-up but not blocking for v1.

## Verdict

**merge-after-nits**

## Rationale

The core design is sound: reused schemas, new IDs on import, transactional restore, project-boundary check on export, todos in the payload to close the original issue. Tests cover the important shapes and the cross-project boundary. The body-limit and streaming concerns are real but reasonable to defer to follow-ups; they don't block the v1 API shape.
