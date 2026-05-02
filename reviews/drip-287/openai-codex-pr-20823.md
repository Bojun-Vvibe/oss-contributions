# openai/codex PR #20823 — Expose structured service tiers in app-server

- Head SHA: `380497d94c8e8ae54dcf942dfc603a00333b4e08`
- URL: https://github.com/openai/codex/pull/20823
- Size: +209 / -8, 12 files
- Verdict: **merge-after-nits**

## What changes

Adds a new `ModelServiceTier { id: ServiceTierId, name: string, description: string }`
schema type to both `codex_app_server_protocol.schemas.json` and
`codex_app_server_protocol.v2.schemas.json`. The existing
`additionalSpeedTiers: string[]` field on `ApiModel` is marked
`deprecated: true` with a description pointing at the new
`serviceTiers: ModelServiceTier[]` field. `ServiceTierId` is introduced
as a string-typed alias.

## What looks good

- Backward-compat is handled correctly: `additionalSpeedTiers` keeps its
  `default: []` and `deprecated: true` flag rather than being removed
  outright. Clients on older builds keep working through one release
  cycle.
- The new `ModelServiceTier` shape (id + display name + description) is
  the minimum viable structure to drive a TUI picker — having all three
  required (`required: ["description","id","name"]`) prevents servers
  from emitting half-populated entries.
- Schema generation is consistent across both v1 and v2 schema files
  (the diff hits both and adds the type in both `definitions/v2/` and
  the v2 root). That keeps generated TS/Rust clients in sync.

## Nits

1. `ServiceTierId` is defined as bare `{"type": "string"}` with no
   `enum` constraint. If the set of valid IDs is closed (e.g.
   `priority`, `default`, `flex`), an `enum` would prevent typos in
   server emissions and let codegen produce a discriminated union on
   the client. If it's intentionally open (so providers can register
   custom tiers), add a one-liner comment in the schema description
   saying so.
2. The deprecation description reads
   `"Deprecated: use \`serviceTiers\` for structured service-tier metadata.\n\n@deprecated use \`serviceTiers\` instead."`
   — the `@deprecated` JSDoc tag duplicates the `deprecated: true` flag
   *and* duplicates the leading "Deprecated:" prose. Pick one canonical
   marker; some codegen tools render all three and produce ugly comments.
3. Confirm `ApiModel.serviceTiers` defaults to `[]` server-side too.
   Schema says `default: []` but if the Rust side serializes
   `Option<Vec<ModelServiceTier>>` as `null`, clients with
   non-null-tolerant codegen will crash.

## Risk

Low — additive schema change with deprecation. Worth merging before
#20824 (the TUI consumer) so clients can roll out independently.
