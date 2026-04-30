# sst/opencode #25035 — refactor: use Effect config for HttpApi authorization

- **URL:** https://github.com/sst/opencode/pull/25035
- **Head SHA:** `631e5d188e8efe31986559d5bd6b8748e276cb87`
- **Files:** `packages/opencode/src/effect/config-service.ts` (new, +67), `packages/opencode/src/server/routes/instance/httpapi/middleware/authorization.ts` (+34/-30), `packages/opencode/src/server/routes/instance/httpapi/server.ts` (+2/-2), `packages/opencode/test/server/httpapi-authorization.test.ts` (+16/-48)
- **Verdict:** `merge-as-is`

## What changed

Replaces module-mutable global env reads (`Flag.OPENCODE_SERVER_PASSWORD`, `Flag.OPENCODE_SERVER_USERNAME`) inside the HttpApi authorization middleware with a typed Effect `ConfigService`-derived layer.

1. **New `ConfigService.Service<Self>()(id, fields)` factory** at `effect/config-service.ts:48-64`. Generates a `Context.Service`-tagged class with two static helpers — `layer(input)` (succeed-from-explicit-input, for tests) and `defaultLayer` (parse from active `ConfigProvider`, for production). The implementation defers to `Config.all(fields).asEffect().pipe(Effect.map((config) => this.of(config as Shape<Fields>)))` at `:60-63`, which is the canonical Effect pattern: keep `Config` as the source of truth for env names + defaults + validation, then wrap it as a service so consumers get DI without touching `process.env`.

2. **`ServerAuthConfig` defined symmetrically** in `authorization.ts:20-26`:
   ```ts
   class ServerAuthConfig extends ConfigService.Service<ServerAuthConfig>()(
     "@opencode/ExperimentalHttpApiServerAuthConfig",
     { password: Config.string("OPENCODE_SERVER_PASSWORD").pipe(Config.option),
       username: Config.string("OPENCODE_SERVER_USERNAME").pipe(Config.withDefault("opencode")) },
   ) {}
   ```
   Password is `Option`, username defaults to `"opencode"` — same semantics the old `Flag`-based code expressed in two ad-hoc places.

3. **`validateCredential` now takes `config` as a third parameter** at `authorization.ts:34-50`. The `if (Option.isNone(config.password) || config.password.value === "")` short-circuit at `:38` preserves the previous "no password configured → bypass auth" behavior, but now the bypass is a function of injected service state rather than a global env read. Errors switched from a custom `Unauthorized` Schema-tagged error to the platform `HttpApiError.Unauthorized({})` / `HttpApiError.UnauthorizedNoContent`.

4. **`authorizationLayer` is now `Layer.effect`** at `:140-150` (was `Layer.succeed`). Pulls `ServerAuthConfig` from context and returns the `Authorization.of({...})` instance closure-captured against that config. Wired in `server.ts:99` as `authorizationLayer.pipe(Layer.provide(ServerAuthConfig.defaultLayer))`.

5. **Tests collapse from 48 lines of `useAuth` ContextVar dance to three flat layers** — `noAuthLayer`, `secretLayer`, `kitSecretLayer` at `httpapi-authorization.test.ts:30-32`. Three `testEffect` instances pre-bind each. Each `it.live` block is now ~10 lines shorter because there's no `yield* useAuth({...})` setup-and-finalize ceremony.

## Why it's right

- **Removes a real mutable-global testability hazard.** The previous test file at the deleted `testStateLayer` / `useAuth` shape (lines 26-72 in the old file) was *literally* mutating `Flag.OPENCODE_SERVER_PASSWORD = ...` and restoring it in a finalizer. Three problems: parallel test runs would race on the global; a finalizer that didn't fire (e.g. on a hard panic) leaked state across files; and the `useAuth` ceremony lived in every test, mixing fixture concerns with assertion concerns. Layer-injection at the test entrypoint solves all three.
- **The factory is a real abstraction, not just sugar.** `ConfigService.Service<Self>()(id, fields)` generates *both* a typed class *and* the `defaultLayer` parser in one declaration. Without it, every config-bearing service in this codebase would re-implement the `Layer.effect(this, Config.all(fields).asEffect().pipe(Effect.map(this.of)))` boilerplate. The factory pattern lets the next config service (LSP config, tunnel config, MCP config) be three lines.
- **`Config.option` vs `Config.withDefault` is the correct asymmetry.** Password has *no* default and being unset means "auth disabled" — `Option<string>` captures that. Username has a sensible default (`"opencode"`) and being unset means "use the default" — `withDefault` captures that. The previous `Flag.OPENCODE_SERVER_USERNAME ?? "opencode"` at `:31` was the same intent expressed in a less-typed shape.
- **`HttpApiError.UnauthorizedNoContent` switch is a small win.** The custom `Unauthorized` schema with a hardcoded `message: "Unauthorized"` string was redundant — clients only get a 401 status, body is never read. Using the platform-supplied error removes the schema definition and is one fewer thing for a future maintainer to keep in sync.
- **`Context.makeUnsafe<unknown>(new Map())` at `server.ts:59`.** Replaces `Context.empty() as Context.Context<unknown>`. This is incidental cleanup — `Context.empty()` returns `Context.Context<never>` and the cast was a code-smell; `makeUnsafe` is the documented escape hatch when the empty-context-as-unknown trick is needed.

## Tiny nits (non-blocking)

- The new `ConfigService.Service` factory uses `oxlint-disable-next-line typescript-eslint/no-unsafe-type-assertion` twice (`config-service.ts:53` and `:62`). Both are justified by the comments, but a follow-up exploring whether `Config.all`'s overload signature can be narrowed to remove the inner cast at `:62` would be a nice cleanup.
- The new factory lives at `src/effect/config-service.ts`. The `export * as ConfigService from "./config-service"` at `:67` is the namespace-re-export pattern, but there's no barrel `src/effect/index.ts` shown here. If callers `import { ConfigService } from "@/effect/config-service"` everywhere (as `authorization.ts:1` does), that's fine; if a barrel exists elsewhere and needs an entry added, that's a follow-up touch.
- `ServerAuthConfig.defaultLayer` is provided at the layer-merge site (`server.ts:99`). If another middleware ever needs the same config, the layer instance gets duplicated — `Layer.fresh` semantics would re-parse `Config`. In practice `Config` parsing is pure and idempotent, so this is theoretical, but a `memoMap`-cached singleton (the file already imports `memoMap` at `server.ts:57`) would be belt-and-braces.

## Risk

Low. The behavior change is invisible to clients: same env vars, same defaults, same 401-on-bad-credentials. The only observable difference is the 401 body — previously a JSON-encoded `{ "_tag": "Unauthorized", "message": "Unauthorized" }`, now whatever `HttpApiError.UnauthorizedNoContent` emits (likely empty or a minimal RFC7807 shape). Any client that was parsing the previous body would notice; clients that just check `response.status === 401` are unaffected. The test suite covers basic, basic-with-username, auth-token, and malformed-token paths, all of which pass against the new shape.
