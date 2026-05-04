# openai/codex#21012 — memories/mcp: generate tool schemas with schemars

- PR ref: `openai/codex#21012`
- Head SHA: `613f90fc6426c1982aaa31c8c878c866c7d6f022`
- Title: memories/mcp: generate tool schemas with schemars
- Verdict: **merge-after-nits**

## Review

This is a very satisfying refactor: the four hand-rolled `serde_json::json!()` schema
builders in `codex-rs/memories/mcp/src/schema.rs` (the entire pre-PR file, ~136 lines)
collapse to a single generic `schema_for::<T>()` plus thin `input_schema_for` /
`output_schema_for` wrappers (`schema.rs:1-42`). Backing that with
`#[derive(JsonSchema)]` plus `#[schemars(deny_unknown_fields)]` on each request /
response struct (`backend.rs:42`, `:59`, `:79`, `:91`, `:99`, `:106`) means the JSON
Schema and the Rust types can no longer drift — exactly the right invariant for an
MCP tool surface.

The split between `option_add_null_type: false` for inputs and `true` for outputs
(`schema.rs:11-14`) deserves the inline comment it already has — that's the standard
schemars convention to keep `Option<T>` permissive on the way in but fully described
on the way out, and it matches what the previous hand-written schemas were doing
implicitly with `anyOf: [string, null]`.

Nit: pulling `schemars 0.8.22` in directly (`Cargo.toml:22`, `Cargo.lock:3050`) while
`rmcp` already re-exports schemars via its `"schemars"` feature is slightly redundant
— it works because both resolve to the same version, but a future rmcp bump that
moves to schemars 1.x will produce a confusing two-version-in-tree error. Worth a
`# keep in lockstep with rmcp's schemars feature` comment in `Cargo.toml`. Otherwise
the diff is a strict improvement and the `into_root_schema_for` + `inline_subschemas`
choice is correct for keeping the emitted schema flat (no `$defs` indirection that
some MCP clients still struggle with).
