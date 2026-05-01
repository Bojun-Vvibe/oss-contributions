# PR #25291 — fix(httpapi): preserve OpenAPI parameter parity

- Repo: sst/opencode
- Head: `1e69246530b34a0c6c8ce04589020184b2feeac4`
- URL: https://github.com/sst/opencode/pull/25291
- Verdict: **merge-as-is**

## What lands

The Effect-based `httpapi/` group is migrating routes off the legacy Hono
surface. The `OPENCODE_SDK_OPENAPI=httpapi` SDK build was producing parameter
schemas that diverged from the legacy spec for ID-shaped path/query params
(missing the `^ses.*` / `^msg.*` / `^prt.*` patterns) and the parity bridge
test wasn't catching it because `parameterKey()` only hashed
`in:name:required` without the schema. This PR closes both halves.

## Specific findings

- `packages/opencode/src/server/routes/instance/httpapi/public.ts:80-87` adds
  `PathParameterSchemas` for the five ID prefixes (`ses`/`msg`/`prt`/`per`/
  `pty`). The lookup table is the right shape — each entry is just `{type:
  "string", pattern: "^xxx.*"}` and there's a focused fall-through
  `pathParameterSchema()` (`public.ts:514-522`) for the four `id`/`requestID`
  cases that don't fit a single static map (workspace `wrk*`, permission
  `per*`, question `que*`, scoped by HTTP method + route prefix).
- `public.ts:494-502` also fixes a real query-side gap: the previous
  `normalizeParameter` early-returned for non-query params (`if (param.in
  !== "query" || ...) return`), which silently let path params keep the
  Effect-generated `allOf`/optional-null junk. Now path params route through
  either the override or `stripOptionalNull()`.
- `public.ts:441-446` extends `stripOptionalNull` with a one-arm `allOf`
  collapse — when Effect emits `{allOf: [constraint]}`, it merges the
  constraint into the parent and recurses. Correct semantics: a single-arm
  intersection IS the constraint.
- The bridge test at `test/server/httpapi-bridge.test.ts:121-138` now hashes
  the schema into the parity key via `stableSchema()` — sorts object keys
  recursively before `JSON.stringify`, so `{type:"string",pattern:"..."}`
  and `{pattern:"...",type:"string"}` collapse to the same key. Without
  this, the kind of regression this PR fixes would never have been caught
  by the bridge test.

## Verdict rationale: merge-as-is

- The new `AGENTS.md` at `packages/opencode/src/server/routes/instance/AGENTS.md:1`
  documents exactly the constraint that was being violated ("legacy Hono and
  Effect HttpApi must stay behaviorally aligned") so future drift gets caught
  at code-review time, not just by the bridge test.
- The path-parameter override carve-out (`requestID` differs between
  `/permission/` → `^per.*` and `/question/` → `^que.*`) is keyed on the
  HTTP method + route prefix, which matches how `OpenCodeHttpApi` actually
  groups those endpoints — no risk of one prefix accidentally claiming the
  other.
- Validation footprint is large and concrete (six httpapi test files +
  direct OpenAPI comparison + SDK regen). The PR author explicitly notes
  generated SDK files were inspected and restored, so this is a
  source-only diff.
- The `stripOptionalNull` allOf-collapse is the one piece that could
  theoretically regress something — but it's gated on `allOf.length === 1`,
  so multi-arm intersections (which have legitimate semantics) are
  untouched.
