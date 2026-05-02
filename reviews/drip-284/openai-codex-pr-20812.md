# Review: openai/codex#20812

- **PR:** openai/codex#20812
- **Head SHA:** `c76e96986d40734c68eea1667d0a0eb368dace33`
- **Title:** Use backend service-tier metadata in app-server and TUI
- **Author:** aibrahim-oai

## Files touched (high-level)

- `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol{,.v2}.schemas.json` and `codex-rs/app-server-protocol/schema/json/v2/ModelListResponse.json` — change `Model.additionalSpeedTiers: string[]` → `Model.serviceTiers: ModelServiceTier[]`, where each element is `{ id: ServiceTier, name: string, description: string }`.
- `codex-rs/app-server-protocol/schema/typescript/ServiceTier.ts` — relax the type from `"fast" | "flex"` to plain `string`.
- `codex-rs/app-server-protocol/schema/typescript/v2/Model.ts` — regenerate the `Model` shape to match.
- (Implied by the schema changes) corresponding Rust struct rename in `codex-rs/app-server-protocol/src/protocol/v2.rs` and TUI/app-server code that consumes the field.

## Specific observations

- `ServiceTier.ts`: `export type ServiceTier = "fast" | "flex"` → `export type ServiceTier = string`. This widens the type to "any string the backend returns" with a documented set of known values (`priority`, `flex`, `ultrafast`). The schema description acknowledges this: _"String-backed service tier identifier. Known values include `priority`, `flex`, and `ultrafast`, but other backend-provided ids are allowed."_ This is a deliberate move from a closed enum to a forward-compatible open string, which is the right call for a backend-provided identifier — but it removes compile-time exhaustiveness in TS callers. Worth a brief search for `switch (tier)` / `tier === "fast"` in TUI code, since `"fast"` is no longer in the documented set (only `priority|flex|ultrafast`); any holdover branch on `"fast"` will silently never match.
- `Model.ts` / schemas: `additionalSpeedTiers: string[]` → `serviceTiers: ModelServiceTier[]`. This is a **breaking rename** of a top-level field on `Model`. Older clients reading `additionalSpeedTiers` will get `undefined`. If this protocol version has any external consumers (VS Code extension, Anthropic-compatible MCP wrappers, etc.), they need a coordinated bump. The new shape is strictly richer (id + name + description), so the migration target is clear, but there is no aliasing/back-compat in the diff.
- `ModelServiceTier` is `{ id: ServiceTier, name: string, description: string }` with all three fields required. Sensible — backend always provides display name + description, no `Option<String>` shenanigans.
- The diff also fixes the trailing-newline issue on `codex_app_server_protocol.schemas.json` and `codex_app_server_protocol.v2.schemas.json` ("\ No newline at end of file" → newline added). Cosmetic, fine.
- The cross-PR coupling with #20815 (also touching `protocol/v2.rs`) is worth noting for the merge order — likely no actual conflict since they're different fields, but rebasers should expect a generated-schema reroll either way.

## Verdict

**needs-discussion**

## Reasoning

The intent is clean (replace a closed `additionalSpeedTiers: string[]` enum with a richer `serviceTiers: ModelServiceTier[]` array carrying display metadata) and the schema regeneration looks consistent. But this is a **breaking field rename** on a publicly-exposed protocol surface, plus it relaxes the `ServiceTier` enum from `"fast" | "flex"` to open `string` while changing the documented set of known values to `priority | flex | ultrafast` — `"fast"` quietly disappears. Before merge, the team should: (1) confirm no in-tree TS code branches on the legacy `"fast"` literal or reads the legacy `additionalSpeedTiers` field, (2) decide whether the protocol bump warrants a `v2` → `v3` jump or an alias period, and (3) document the migration path for downstream clients. Mechanically correct, just not a "merge silently" change.
