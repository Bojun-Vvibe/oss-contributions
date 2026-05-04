# Review: openai/codex #20969

- **Title:** 1- Add model service tiers metadata
- **Head SHA:** `b59bce8863401725d24ec054b2fb613dff6c8abe`
- **Size:** +190 / -4
- **Files touched:**
  - `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.schemas.json`
  - `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.v2.schemas.json`
  - `codex-rs/app-server-protocol/schema/json/v2/ModelListResponse.json`
  - `codex-rs/app-server-protocol/schema/typescript/v2/Model.ts`
  - `codex-rs/app-server-protocol/schema/typescript/v2/ModelServiceTier.ts`
  - `codex-rs/app-server-protocol/schema/typescript/v2/index.ts`
  - `codex-rs/app-server-protocol/src/protocol/v2.rs`
  - `codex-rs/app-server/src/models.rs`

## Critique

Introduces a new `ModelServiceTier { id, name, description }` struct to the v2 schema and adds a `serviceTiers: ModelServiceTier[]` field on `Model`, while marking the older `additionalSpeedTiers: string[]` as deprecated via JSON-schema description.

Specific lines:

- `codex_app_server_protocol.schemas.json:11215` and `.v2.schemas.json:7868` and `v2/ModelListResponse.json:27` — all three add `"description": "Deprecated: use \`serviceTiers\` instead."` to `additionalSpeedTiers`. Consistent. JSON Schema doesn't have a first-class `deprecated` flag (well, draft-2019-09 does), so a description-only deprecation is reasonable but won't trigger codegen warnings. If this schema dialect supports `"deprecated": true`, prefer that for future-proofing.
- `codex_app_server_protocol.schemas.json:11261-11267` — new `serviceTiers` field with `default: []`. Good — keeps backward compat for clients that don't send it.
- `ModelServiceTier` struct (line 11432-11448 in `.schemas.json`, lines 8085-8101 in `.v2.schemas.json`, lines 130-150 in `v2/ModelListResponse.json`) — all three required fields (`description`, `id`, `name`). Confirm in the Rust source `v2.rs` that the struct matches the same required-set, otherwise serde will silently produce different shapes.
- This is PR `1-` of what is presumably a series ("1- Add model service tiers metadata"). The numeric prefix in the title is unusual; downstream PRs likely consume these fields. As a standalone change it's a pure additive schema bump and should not break existing v2 clients.
- The TypeScript files (`Model.ts`, `ModelServiceTier.ts`, `index.ts`) are presumably generated. Verify they were regenerated from the schema and not hand-edited (a quick CI step or `cargo run -p schema-gen` would confirm).

Backward compatibility: additive only (new optional field defaulted to `[]`, new struct). Low risk.

No banned strings.

## Verdict

`merge-as-is`
