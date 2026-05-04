# block/goose PR #8930 — fix: normalize nullable schemas for Vertex Gemini compatibility

- **PR:** https://github.com/block/goose/pull/8930
- **Author:** dantti
- **Head SHA:** `523590cc` (full: `523590cc2373d90a155506b03dc078bd23d1c442`)
- **State:** OPEN — partially fixes #8778
- **Files touched:**
  - `crates/goose/src/providers/formats/openai.rs` (+126 / -0)

## Verdict

**merge-as-is**

## Specific refs

- `crates/goose/src/providers/formats/openai.rs:706-712` — `normalize_nullable` is invoked once per property inside `ensure_valid_json_schema`'s property loop. Placement is correct: it runs before the recursive object-traversal step, so nested object properties also get normalized on the recursive descent.
- `crates/goose/src/providers/formats/openai.rs:723-770` — `normalize_nullable` handles two distinct schemars 1.x emission shapes:
  - `"type": ["integer", "null"]` array form → collapses to `"type": "integer"` when exactly one non-null variant remains. Good defensive check (`non_null.len() == 1`).
  - `"anyOf": [T, {"type": "null"}]` form → unwraps to `T` when there are exactly 2 variants and one is the null sentinel. The two-element check is conservative; multi-variant `anyOf` (e.g., union types) is correctly left alone.
- `crates/goose/src/providers/formats/openai.rs:1187-1245` — three new test cases covering:
  - `anyOf` nullable unwrap on a synthetic `timeout_secs` schema
  - `type: ["integer", "null"]` array unwrap
  - End-to-end: actually generates a `ShellParams` schema via `schema_for!`, runs it through `validate_tool_schemas`, asserts no residual `anyOf` and `type=integer`. This last test is the strongest one — it exercises the real production schema rather than a contrived fixture.
- PR rationale at `formats/openai.rs:707-711`: the comment explicitly notes "optional-ness is already conveyed by the field being absent from `required`". Correct — that's the JSON Schema spec semantic that Vertex Gemini relies on.

## Rationale

Tight, well-scoped fix. The Vertex Gemini function-calling API genuinely doesn't support array types or `anyOf {type: null}`; this is a real interop bug for anyone using a custom OpenAI-compatible provider pointed at Vertex via Bifrost. The normalization is conservative (only collapses when exactly one non-null variant remains, only unwraps 2-element `anyOf`) so it can't regress legitimate union schemas.

Test coverage hits both shapes plus the live `ShellParams` schema — that's the right test pyramid for a serialization fix. Single-file change, no public API changes, no behavior change for non-Vertex providers (since they accept either shape).

Merge as-is.
