# block/goose #8785 — Add Location column to CLI skills table

- **PR:** https://github.com/block/goose/pull/8785
- **Head SHA:** e048aa8ead15eeff2121f3308feed5bd0cef1e8a
- **Files changed:** 1 file, +14 / −2 (`crates/goose-cli/src/session/mod.rs`)
- **Verdict:** `merge-after-nits`

## What it does

Adds a third column to the `goose skills list` ASCII table so the operator can tell at
a glance whether a skill is shipped with the binary, installed globally for the user,
or scoped to the current project. Three-way classification at
`session/mod.rs:931-937`:

```rust
let location = if skill.source_type == SourceType::BuiltinSkill {
    "built-in"
} else if skill.global {
    "global"
} else {
    "project"
};
```

Header expanded from `["Skill", "Description"]` to `["Skill", "Location",
"Description"]` (`session/mod.rs:923`), and each row now carries the new cell.
Imports `goose::custom_requests::SourceType` (line 910).

## What's good

- Genuinely useful: identical-name shadowing (built-in `<skill>` overridden by a
  global `<skill>` overridden by a project `<skill>`) was previously invisible from
  the listing; this surfaces it without a verbose flag.
- Three-state classification is exhaustive against the current `SourceType` /
  `global` shape — no silent fallthrough.
- The change uses the same `comfy_table` `ContentArrangement::Dynamic` so column
  widths still adapt to terminal width.

## Nits / risks

1. The classification ladder treats `(SourceType != BuiltinSkill) && global == true`
   as `"global"`, regardless of which non-builtin source type it is. If `SourceType`
   later grows a `RemoteSkill` or `MarketplaceSkill` variant, the listing will
   misreport it as `"global"` or `"project"` based purely on the boolean. Worth a
   `match skill.source_type { ... }` shape now (or at least a `// TODO if SourceType
   gains variants`) so the next contributor sees the audit point.
2. No test added. A 4-row golden test of `handle_list_skills` against a fixture set
   covering all three location values would lock the contract; the existing skills
   test harness should make this trivial.
3. Optional copy nit: column value `"built-in"` mixes with `"global"` / `"project"`
   stylistically — consider `"builtin"` (no hyphen) for consistency, or capitalize
   all three.

## What I learned

`comfy_table` makes appending a column cheap, but the real design question is the
classification function — once `Location` is shown, every future `SourceType`
variant becomes a UX decision, not just a backend concern. Lock in a `match` now
so the compiler reminds you.
