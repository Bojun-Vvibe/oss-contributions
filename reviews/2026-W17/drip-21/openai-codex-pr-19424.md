# openai/codex#19424 — Strip sandbox access defaults from app-server JSON schema

- **PR:** https://github.com/openai/codex/pull/19424
- **Head SHA:** `0104371752ec453063414fea8868eab979a7c397`
- **Files:** 8 schema fixtures + `codex-rs/app-server-protocol/src/export.rs` (+88/-64)
- **Verdict:** **merge-after-nits**

## Context

Continues the `SandboxPolicy → PermissionProfile` migration that has
dominated the recent codex stack (see drip-19, drip-20). Specifically
this PR cleans up the **JSON-schema export** for the
`ReadOnlySandboxPolicy.access` and `WorkspaceWriteSandboxPolicy.readOnlyAccess`
fields. Both fields currently emit a `default: { "type": "fullAccess" }`
into the generated app-server schemas, which is a codegen-hostile
shape: most JSON-schema-to-TypeScript / -Python / -Rust generators
either choke on object-valued defaults inside `oneOf` or emit a
literal that diverges from the runtime `serde` default.

## Problem

`ReadOnlyAccess` is a tagged union (`{ "type": "fullAccess" }` vs
`{ "type": "denyRead", "paths": [...] }`). At the schema level this
is a `oneOf` over two `$ref`s, and the generated default —
`{ "type": "fullAccess" }` — is currently emitted **alongside** the
`oneOf`, e.g. (`ClientRequest.json:3061-3066`, before):

```jsonc
"access": {
  "oneOf": [
    { "$ref": "#/definitions/FullAccess" },
    { "$ref": "#/definitions/ReadOnlyAccess" }
  ],
  "default": { "type": "fullAccess" }
}
```

Two practical issues with that:

1. Object-valued `default` inside `oneOf` is poorly handled by most
   downstream generators: some emit a literal (`{type: "fullAccess"}`
   typed as `unknown`), some emit nothing, and some refuse to
   generate the parent type altogether. The runtime `serde` default
   is the canonical source of truth — emitting it into the schema
   adds two contracts that can drift.
2. Boundary symmetry: scalar defaults like `networkAccess: false` are
   useful (consumers can drop the field on the wire and the receiver
   round-trips faithfully). Object defaults masquerade as the same
   thing but actually require any client to construct the same nested
   object on the wire to round-trip — there's no value to advertising
   them.

## Design

The fix is small and well-targeted. `export.rs` adds a
`strip_schema_property_default(value, schema_title, property_name)`
helper that walks the produced `Value::Object` and removes the
`default` key from `properties.<property_name>` if and only if the
schema's `title` matches `schema_title`. It's then called twice,
inside the per-file post-processing pipeline that already runs
`enforce_numbered_definition_collision_overrides` and
`annotate_schema`:

```rust
strip_schema_property_default(&mut schema_value, "ReadOnlySandboxPolicy", "access");
strip_schema_property_default(
    &mut schema_value,
    "WorkspaceWriteSandboxPolicy",
    "readOnlyAccess",
);
```

The fixture diffs are mechanical and uniform — every one of the 8
generated schemas drops the same 4-line `"default": { "type":
"fullAccess" }` block from each of the two locations. The scalar
`"default": false` for `networkAccess` is preserved (visible in the
unchanged diff context), so the carve-out is correctly scoped.

## Strengths

- **Symmetry decision is explicit and defensible.** The PR body
  spells out *"keep runtime serde defaults intact and leave scalar
  schema defaults such as `networkAccess: false` untouched"* — that's
  exactly the right line to draw, because scalar defaults are
  cheap-to-honor at the wire boundary and object defaults aren't.
- **Helper is reusable.** `strip_schema_property_default` taking
  `(schema_title, property_name)` is generic enough that future
  similar carve-outs (e.g. if other tagged unions get the same
  treatment) get a one-liner instead of more bespoke fixture
  surgery.
- **Mechanical fixture refresh** is good hygiene — no ad-hoc edits to
  the JSON files; everything flows from the new exporter logic.

## Risks / nits

1. **Title matching is name-coupled.** The carve-out keys on
   `schema_title == "ReadOnlySandboxPolicy"` /
   `"WorkspaceWriteSandboxPolicy"`. A future rename of either type
   silently turns the carve-out into a no-op and the
   object-valued defaults will reappear in the next regenerated
   fixture, with no test signal. The unit tests aren't shown in the
   visible portion of the diff — a regression test that asserts
   *"none of the generated schemas contain the literal
   `\"type\": \"fullAccess\"` as a `default` value"* would close
   this gap cheaply.
2. **No explicit assertion about scalar default preservation.** The
   author claims `networkAccess: false` is left intact, and the
   diffs confirm it, but a one-line test would lock that in too.
3. **`strip_schema_property_default` signature.** Taking
   `(value, schema_title, property_name)` is fine; consider also
   taking the property as a slice (`&[&str]`) so a single helper can
   strip multiple defaults from the same schema in one walk if
   future use requires it. Minor, not a blocker.
4. The new function lives directly in `export.rs`. If more
   schema-shaping helpers accumulate, it'd be worth carving them
   into a `schema_postprocess` module. Not for this PR.

## Verdict — merge-after-nits

The behavioral change is exactly the right shape: shrink the schema
contract to what's actually round-trippable on the wire, leave the
runtime serde default as the single source of truth. Worth landing
behind one regression test that pins both invariants (no object
default reappears; scalar `networkAccess: false` stays).

## What I learned

JSON-schema export pipelines that publish *both* a `oneOf` and a
matching object-valued `default` accumulate a class of subtle
contract drift: downstream generators handle the combination
inconsistently, and reviewers tend to ignore "fixture-only" diffs
because they're noisy. The pattern of *"add a small post-processing
helper keyed on schema-title-and-property-name, and let the fixtures
flow from it"* is the right shape for these cleanups — but the
trade-off is that the carve-out becomes invisible to renames. A
single negative regression test (*"no generated schema contains X"*)
is much cheaper than re-discovering this in a year when an unrelated
type rename quietly reintroduces the bug.
