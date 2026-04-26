# sst/opencode PR #24459 — feat(opencode): add async command endpoint

- **PR:** https://github.com/sst/opencode/pull/24459
- **Head SHA:** `4dcbef32a1ce49cb808ffd37814821b352c7d336`
- **Files:** 4 (+154 / -4)
- **Verdict:** `merge-after-nits`

## What it does

Adds `POST /session/:sessionID/command_async` mirroring the existing async session
routes. Handler at `packages/opencode/src/server/routes/instance/session.ts:965-1003`
delegates to `SessionPrompt.Service.command(...)` via `runRequest(...)`, returns
`204 No Content` immediately, and on background failure publishes
`Session.Event.Error` with a `NamedError.Unknown`. Compression middleware at
`packages/opencode/src/server/middleware.ts:90` is updated to also exclude the
new route from gzip (correct — async endpoints have empty bodies, gzip is a
small CPU hit for nothing). SDK `gen/sdk.gen.ts` adds a `commandAsync(...)`
method and `gen/types.gen.ts` adds the matching `SessionCommandAsync*` type
quartet.

## Specific reads

- `middleware.ts:90` — regex now `^/session/[^/]+/(message|prompt_async|command_async)$`. Pattern is consistent with the `prompt_async` precedent.
- `session.ts:984-997` — `void runRequest(...).catch(err => log.error + Bus.publish(Session.Event.Error, ...))`. The `void` keyword is intentional and necessary so the Hono handler isn't accidentally `await`ed. Good.
- `session.ts:990` — error is wrapped with `err instanceof Error ? err.message : String(err)`. Pragmatic.
- The two `NamedError` import paths (`session.ts:28`, `middleware.ts:2`) silently switch from `@opencode-ai/core/util/error` → `@opencode-ai/shared/util/error`. That's a drive-by import-path migration that should ideally land in a separate refactor commit, not folded into a feature PR — easy to miss in review and harder to revert.

## Nits before merge

1. **Drive-by import migration**: split the two `NamedError` import-path swaps and the `Flag` import swap (`middleware.ts:11` → `@/flag/flag`) out into their own chore commit so the feature is reviewable in isolation.
2. **No test added**: the existing `prompt_async` route presumably has a test that asserts (a) returns `204` quickly, (b) error from background work surfaces via `Session.Event.Error`. Mirror it for `command_async` — both branches are critical and currently untested.
3. **`runRequest`'s second arg is `c`**: the Hono `Context` is captured into a fire-and-forget background task. Confirm `c` doesn't hold request-scoped state (e.g. an aborted `AbortSignal`) that becomes invalid the moment the handler returns `204`. If `runRequest` reads anything off `c.req` after the handler exits, the request body may already be GC'd.
4. **OpenAPI `204` vs error events**: clients that get `204` then receive a delayed `Session.Event.Error` need a way to correlate the two. Document in the route description that callers should subscribe to `Session.Event.Error` filtered by `sessionID` *before* posting, or include a `messageID` in the response body (still `204` as a header-only ack with `Location:` is also a pattern worth considering).
5. **`compression` carve-out comment**: at `middleware.ts:90` add an inline comment that the route returns `204` with no body so gzip is wasted work, mirroring whatever's documented for `prompt_async`. Otherwise the regex grows silently.
