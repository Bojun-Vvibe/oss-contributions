# sst/opencode PR #24486 — bridge session lifecycle routes onto HttpApi

- **PR**: https://github.com/sst/opencode/pull/24486
- **Author**: @kitlangton
- **Head SHA**: `e50aab55d3193bf93e6d5019548eba68c4c8d27e`
- **Size**: +214 / −7
- **Files**: `packages/opencode/specs/effect/http-api.md`, `packages/opencode/src/server/routes/instance/httpapi/session.ts`, `packages/opencode/src/server/routes/instance/index.ts`, `packages/opencode/test/server/httpapi-session.test.ts`

## Summary

Continues the slow strangler migration of the legacy Hono session routes onto the schema-checked Effect HttpApi surface. This PR specifically bridges the five mutating session lifecycle endpoints — `POST /session` (create), `DELETE /session/:sessionID`, `PATCH /session/:sessionID` (update title / permission / archived), `POST /session/:sessionID/fork`, and `POST /session/:sessionID/abort` — into `SessionApi`, with new schemas `UpdatePayload` and `ForkPayload` and a paths block that mirrors the read-side endpoints introduced in the prior stack PR.

## Verdict: `merge-after-nits`

Right shape. The PR is tightly scoped to bridge parity (the spec checklist is updated in lockstep, item 9 flips to `[x]`), the schemas are derived from canonical types rather than re-typed (`Struct.omit(Session.ForkInput.fields, ["sessionID"])`), and there's a dedicated test file. Two things keep it short of `merge-as-is`:

1. The `update` payload allows a partial `time.archived: number` patch but doesn't define what an explicit `null` means (un-archive). The legacy Hono route's behavior should be the documented contract before this becomes the canonical surface.
2. The `abort` endpoint's success type isn't visible in the snippet I read; it should be `Schema.Boolean` to match the existing `remove` pattern, not `Session.Info`, since aborting an already-completed run is a no-op that returns a status, not a fresh `Info`.

## Specific references

- `packages/opencode/src/server/routes/instance/httpapi/session.ts:33-44` — `UpdatePayload` declares `title`, `permission`, and `time.archived` as optional. The `time` shape (`Schema.Struct({ archived: Schema.optional(Schema.Number) })`) means clients cannot distinguish "leave archived alone" from "clear archived". For a `PATCH` semantics surface this matters; consider `archived: Schema.NullishOr(Schema.Number)` and document that `null` un-archives.
- `packages/opencode/src/server/routes/instance/httpapi/session.ts:45-47` — `ForkPayload` derives from `Session.ForkInput` via `Struct.omit(..., ["sessionID"])`. Good — the `sessionID` is in the URL path, so omitting it from the body avoids the "two sources of truth" footgun. This pattern should be extracted into a small helper (`omitFields(Session.ForkInput, "sessionID")`) and reused for the next stack PR (share/summary/message mutations) instead of being open-coded each time.
- `packages/opencode/src/server/routes/instance/httpapi/session.ts:55-57` — `SessionPaths.create = root` and `SessionPaths.list = root` collide on the same string. That's correct for HTTP (different verbs) but the `create:` / `list:` keys appearing twice in one constant object is fragile if anyone later reorders or switches to a `Record` type. A trivial comment `// same path as list — different verb` would prevent a future refactor from collapsing them.
- `packages/opencode/src/server/routes/instance/httpapi/session.ts:147-181` — five new `HttpApiEndpoint.{post,delete,patch}(...)` calls, each with `OpenApi.annotations({ identifier, summary, description })`. Identifiers are well-namespaced (`session.create`, `session.delete`, `session.update`, `session.fork`, `session.abort`). The descriptions read fine; `session.delete` correctly notes the cascade ("permanently remove all associated data"), which is exactly the kind of contract surface OpenAPI consumers need.
- `packages/opencode/specs/effect/http-api.md:294-296` and `:358` — checklist items for the five lifecycle endpoints flip from `[ ]` to `[x]`, and the implementation-order item 9 also flips. Spec-and-code-in-the-same-PR is the right discipline; nothing to change here.

## Nits

1. Add a `null`-means-clear case to `UpdatePayload.time.archived` before this surface is locked in (see ref above).
2. Verify `abort` returns `Schema.Boolean` (or a status enum), not `Session.Info`. If it does need to return `Info` (e.g. to propagate the post-abort `revert`/`active` state), call that out in the description.
3. Consider extracting the `Struct.omit(...)` body-derivation pattern into a tiny named helper before the next stack PR replicates it.

## What I learned

The Effect `HttpApi` migration here is a textbook strangler: every PR in the stack flips a fixed set of checkboxes in `specs/effect/http-api.md`, and the review is partly "did the code match the checklist diff?". For agent-IDE servers exposing 50+ routes, that spec-as-progress-board pattern is more honest than a sprint board because the authoritative artifact (the route table) is the same one consumers read. Worth stealing.
