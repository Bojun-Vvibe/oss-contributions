# sst/opencode #24706 — fix(httpapi): preserve mcp oauth error parity

- URL: https://github.com/sst/opencode/pull/24706
- Head SHA: `44397fb5c3ee65eb4618698b4621847dba4addf1`
- Size: +122 / -12 across 4 files
- Verdict: **merge-as-is**

## What the change does

The legacy Hono `/mcp/{name}/auth` and `/mcp/{name}/auth/authenticate` routes
respond to "MCP server does not support OAuth" with HTTP 400 + body
`{"error":"MCP server <name> does not support OAuth"}`. The new
`HttpApi`-based routes were responding to the same precondition with
`HttpApiError.BadRequest` whose default schema is `{}` — silently breaking the
SDK contract for any client that pattern-matches on `error`.

Fix introduces a typed `Schema.ErrorClass` named `UnsupportedOAuthError` with
`{ error: Schema.String }` and `httpApiStatus: 400`
(`packages/opencode/src/server/routes/instance/httpapi/mcp.ts:23-26`), wires
both endpoints to use it as their `error:` schema (`:64`, `:87`), and emits the
matching string in both handlers (`:156`, `:170`) — same wording as the legacy
path so JSON byte-for-byte equality holds.

The legacy router at `packages/opencode/src/server/routes/instance/mcp.ts:11-22`
gets a parallel zod `UnsupportedOAuthError` plus a `unsupportedOAuthErrorResponse`
OpenAPI block, swapping the generic `errors(400, 404)` wrapper for an explicit
`400: unsupportedOAuthErrorResponse` + `errors(404)` so both surfaces
contribute the same `McpUnsupportedOAuthError` schema to the generated OpenAPI
document.

The generated SDK at `packages/sdk/js/src/v2/gen/types.gen.ts:2132-2134` and
`:4910-4914`, `:4988-4992` correctly switches `McpAuthStartErrors[400]` and
`McpAuthAuthenticateErrors[400]` from `BadRequestError` to
`McpUnsupportedOAuthError` with the matching description — verifying the
OpenAPI generator round-tripped the change.

## What is load-bearing

The new test at
`packages/opencode/test/server/httpapi-mcp.test.ts:167-187`
("matches legacy unsupported OAuth error responses") is the spec for the parity
guarantee — it constructs both `app(false)` (legacy) and `app(true)` (HttpApi)
in the same test, fires the same `POST` at `/mcp/demo/auth` and
`/mcp/demo/auth/authenticate`, asserts each individually equals
`{status: 400, body: '{"error":"MCP server demo does not support OAuth"}'}`,
and then asserts `httpapiResponse === legacyResponse`. That last assertion is
what catches future drift — any later refactor that changes either branch's
status code or body wording will fail the parity check, not just the absolute
check.

The `withMcpProject` helper at `:39-66` correctly uses
`fs.makeTempDirectoryScoped` plus `Effect.addFinalizer` for the
`Instance.dispose` call, so the new test does not leak instance state across
the suite. The `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` toggle pattern (saved at
`:14`, restored in `afterEach` at `:80`) is exactly the right shape for a
parity test that has to flip a global feature flag.

## Risk

- The `UnsupportedOAuthError` class identifier `"McpUnsupportedOAuthError"`
  matches the zod `meta({ ref: "McpUnsupportedOAuthError" })` so the
  HttpApi-generated schema and Hono-generated schema collide on the same name
  in the OpenAPI document — that is the desired outcome (one shared type for
  consumers), and the SDK regeneration confirms only one definition lands.
- No behaviour change for callers — same status code, same body, just a typed
  schema instead of `{}`. Idempotent on the wire.

## Recommendation

Ship as is. The test is the spec, parity is asserted directly, SDK is
regenerated to match.
