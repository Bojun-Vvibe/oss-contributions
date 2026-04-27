# sst/opencode #24671 — fix(httpapi): preserve optional session fields

- URL: https://github.com/sst/opencode/pull/24671
- Head SHA: `c3d10af784e68277b15fee5ac756b1331b13b485`
- Diff: +255/-225 across `packages/opencode/src/server/routes/instance/httpapi/session.ts` (+16/-2) and `packages/opencode/test/server/httpapi-session.test.ts` (+239/-223).

## Context / problem

The HttpApi `session.list` endpoint was encoding optional `parentID` and `workspaceID` as `null` rather than omitting them, breaking the legacy session-list JSON shape that clients (including older SDK consumers) parse with strict optional-key semantics. Effect's `Schema.optional` round-trips `undefined`-valued keys as `null`, which is wire-incompatible with the prior shape.

## What the fix does

`session.ts:45-58` introduces a small `omitUndefined<S>(schema)` combinator that wraps `Schema.optionalKey(schema)` with a `decodeTo(Schema.optional(schema), ...)` whose encode side is `SchemaGetter.transformOptional(Option.filter((value) => value !== undefined))` — i.e. on encode, `Some(undefined)` collapses to absent rather than `null`. It then rebuilds a `SessionInfoResponse` via `Session.Info.mapFields(Struct.evolve({ workspaceID: () => omitUndefined(WorkspaceID), parentID: () => omitUndefined(SessionID) }))` and swaps `success: Schema.Array(Session.Info)` → `success: Schema.Array(SessionInfoResponse)` at `:140`. The shape only changes at the wire-encoding layer; `Session.Info` is unchanged for in-memory consumers.

## Specific references

- `session.ts:45-58` — the `omitUndefined` combinator. The `decode: SchemaGetter.passthrough({ strict: false })` half is correct because decoding tolerates either absent-key or `null` (legacy clients can still POST either shape); the `encode` side is the load-bearing piece.
- `session.ts:140` (formerly `:123`) — the `success` swap from `Session.Info` → `SessionInfoResponse`. Only the `list` endpoint is rewrapped; `get`/`create`/etc. are not touched, which is consistent with the PR scope ("preserve optional session list JSON shape") but worth noting as an asymmetry — a single-session `GET /session/:id` will still emit `parentID: null` for root sessions.
- `httpapi-session.test.ts` — full conversion from `bun:test`'s `test` to the local `it` from `../lib/effect` (Effect-based test helper) plus `Effect.promise`-wrapping of `createSession`/`createTextMessage`/`request`/`json`. ~225 lines of mechanical refactor for what is fundamentally a 16-line behavior change.

## Risks / nits

- The test refactor is bundled with the bug fix. ~225 of 239 added test lines are the `bun:test` → effect-test-helper migration, not new coverage. Reviewer attention is split between "does the fix work" and "does the test infra refactor not regress". Splitting into two PRs would make the bug fix trivially reviewable.
- Asymmetry: only `list` is rewrapped. If the wire-shape contract for `parentID`/`workspaceID` is "omitted-not-null", it should hold for `get` too. Either justify the carve-out in the PR body or extend.
- The `omitUndefined` helper is defined inline; if more endpoints need it, it should land in a shared schema-utils module.

## Verdict

`merge-after-nits` — the fix is correct and minimal at the route layer, but the bundled test-infra refactor and the `list`-only carve-out should be addressed (split or justified) before merge.
