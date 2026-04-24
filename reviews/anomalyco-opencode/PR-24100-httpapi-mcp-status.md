# PR-24100 — feat(httpapi): bridge MCP status endpoint

[sst/opencode#24100](https://github.com/sst/opencode/pull/24100)

## Context

Continues the gradual migration of opencode's HTTP surface from the
hand-rolled Hono-on-Zod routes to Effect's `HttpApi` layer behind the
existing experimental flag. This PR:

1. Rewrites `MCP.Status` in `packages/opencode/src/mcp/index.ts` from
   a Zod `discriminatedUnion` of 5 variants
   (`connected`, `disabled`, `failed`, `needs_auth`,
   `needs_client_registration`) to an Effect `Schema.Union` with the
   same 5 `Schema.Struct` variants.
2. Pipes the union through `withStatics((s) => ({ zod: effectZod(s)
   }))`, so legacy callers can still reach the Zod shape via
   `MCP.Status.zod` from the existing Hono routes
   (`server/routes/instance/mcp.ts`).
3. Adds a new `httpapi/mcp.ts` group with a single `GET /mcp` →
   `Schema.Record(Schema.String, MCP.Status)` endpoint, registered
   under the `Authorization` middleware and bridged into the Hono
   app at `instance/index.ts` only when the experimental flag is on.
4. Adds a focused `httpapi-mcp.test.ts`.

## Strengths

- **Two-flavor schema via `withStatics`** is the right shape for an
  incremental migration: the Effect schema is the source of truth,
  and `.zod` is a derived, identifier-preserving compatibility view.
  Keeps the diff at the call sites tiny (just `MCP.Status` →
  `MCP.Status.zod`).
- **OpenAPI annotations on the endpoint** (`identifier:
  "mcp.status"`, `summary`, `description`, `title`, `version`) carry
  through to the generated SDK, so downstream consumers get
  documented types instead of `unknown`.
- **Status-write paths intentionally not migrated.** The PR
  description explicitly defers OAuth/auth flows. That matches the
  surface-area-first sequencing already in `specs/effect/http-api.md`
  — `mcp` flips from `later` to `bridged (partial)` rather than
  attempting a wholesale port.
- **Endpoint is gated behind the experimental flag.** The Hono
  primary route still serves traffic; the HttpApi route is opt-in.
  Low blast radius.

## Concerns / risks

- **Indentation glitch in `mcp.ts`**:
  ```ts
  schema: resolver(MCP.Status.zod),
  ```
  appears with 12-space leading indent inside the
  `oauth-callback`/`status` blocks (visible in the diff:
  `                    schema: resolver(MCP.Status.zod),`). Likely
  a mid-edit mistake; cosmetic but caught by a strict formatter.
- **`identifier` collisions.** The Effect `Schema.Struct` variants
  are renamed identifiers (`MCPStatusConnected`, `MCPStatusDisabled`,
  etc.) to match the Zod `meta({ ref: ... })` calls they replace. If
  `effectZod()` doesn't honor the same `identifier` → `ref`
  translation, the generated OpenAPI / TypeScript types will drift
  from the Hono route's existing schema names. Worth a `bun run
  build` artifact diff to confirm parity.
- **`Status.zod` is a static side-channel.** Callers reach into
  `MCP.Status.zod` to get the Zod shape, but nothing in the type
  system stops them from passing the Effect schema where the Zod
  was expected (or vice versa) — the runtime mismatch is silent
  until a request comes in. A dedicated `MCP.StatusZod` named export
  would make the boundary explicit.
- **Discriminator annotation.** The Union is annotated with
  `discriminator: "status"`. Effect's `Schema.Union` does NOT
  intrinsically use the discriminator for parsing — that requires
  `Schema.Union` with `Schema.TaggedStruct` or
  `Schema.discriminatedBy`. If runtime decoding falls back to
  best-fit matching, an `error: ""` field in a `failed` variant
  could parse as either `failed` or
  `needs_client_registration`. Worth verifying decode behavior with
  a malformed payload test.
- **Test file (`httpapi-mcp.test.ts`) is +48 lines** but the PR
  description doesn't say what it asserts (status round-trip?
  auth middleware? error variants?). Diff was truncated above the
  test file in my fetch; reviewers should confirm the test covers
  at least the `failed` and `needs_client_registration` shapes,
  not just `connected`.

## Verdict

**Approve with caveats.** Migration step is well-scoped and the
`withStatics` bridge is clean. Before merge: fix the indentation
glitch in `mcp.ts`, verify generated SDK types match the previous
Hono-route schema names exactly (no silent rename of `MCPStatus*`
identifiers downstream), and add a decode test for ambiguous
`failed` vs `needs_client_registration` payloads to confirm the
discriminator annotation does what the reader assumes.
