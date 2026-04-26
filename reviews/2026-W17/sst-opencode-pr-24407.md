# sst/opencode #24407 — feat(httpapi): bridge experimental tool routes

- **Repo**: sst/opencode
- **PR**: #24407
- **Author**: kitlangton (Kit Langton)
- **Head SHA**: 2a600797fb0027b3071bab98a5b2419affe639d9
- **Base**: dev
- **Size**: +106 / −6 across 4 files: spec doc, the experimental
  HttpApi handler (+69/−1), instance index (+2/+0), and a test
  (+31/−1).

## What it changes

Bridges two more experimental routes through the typed HttpApi layer
(`packages/opencode/src/server/routes/instance/httpapi/experimental.ts`):

1. `POST /experimental/console/switch` — `ConsoleSwitchPayload =
   { accountID, orgID }` → boolean; under the hood calls
   `account.use(accountID, Some(orgID))` and maps any failure to
   `HttpApiError.BadRequest`.
2. `GET /experimental/tool?provider=&model=` — returns
   `Array<{ id, description, parameters }>` where `parameters` is
   produced via `EffectZod.toJsonSchema(item.parameters)`. Pulls the
   default agent via `agents.get(agents.defaultAgent())` to scope tool
   visibility.

Spec doc (`http-api.md` lines 184, 261-263, 353) is updated to mark
both routes as bridged.

## Strengths

- Mechanical, low-risk addition — both routes already exist in the
  legacy Hono surface; this PR just publishes them via the typed
  HttpApi so they appear in the OpenAPI/SDK output. Symmetric with
  the prior bridges.
- Schema annotations (`identifier: "experimental.console.switchOrg"`,
  `summary`, `description`) are populated, which means the generated
  OpenAPI spec stays self-documenting.
- The `consoleSwitch` handler maps `account.use` failures to
  `BadRequest` rather than letting Effect's default 500 handler
  expose internals. Right error class — bad accountID/orgID is a
  caller error.
- Spec checklist updated in the same PR (lines 261, 263, 353), and
  the partial-bridge note in the table updated to match. Good
  spec-doc hygiene.

## Concerns / asks

- The `tool` handler uses `agents.get(yield* agents.defaultAgent())`.
  If the legacy Hono route allowed callers to specify an agent (or
  used a different default), this is a behavioral change. Confirm
  parity with the legacy implementation. Worth either adding an
  optional `agent` query param or a comment explaining the default
  choice.
- `EffectZod.toJsonSchema(item.parameters)` is called on every
  request. If tools are stable per (provider, model), caching the
  JSON-schema output once at registry build time would avoid the
  per-request schema serialization. Probably a future optimization,
  not blocking.
- `Schema.Record(Schema.String, Schema.Any)` for the `parameters`
  field is loose. JSON Schema has a known shape; declaring a typed
  schema here would catch tool metadata regressions and improve the
  generated SDK type.
- The new test is +31 lines (presumably one round-trip per route).
  Both new routes deserve a happy-path and at least one error-path
  test (e.g. `consoleSwitch` with a bogus accountID returning 400).
  Visible diff slice is too short to confirm.

## Verdict

`merge-after-nits` — clean bridge that follows the existing
pattern. Asks: confirm parity on the agent-default choice for
`/experimental/tool`, tighten the parameters schema, and ensure
both routes have a 400-path test.
