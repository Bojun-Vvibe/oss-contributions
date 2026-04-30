# block/goose#8934 — feat(recipe): support structured parameters (object/array) in recipe templates

- PR: https://github.com/block/goose/pull/8934
- Head SHA: `adb49849c6fa93d8865d898e0eea894a85ad29c6`
- Author: jordigilh (Jordi Gil)
- Files: `crates/goose/src/recipe/mod.rs` +134/0,
  `crates/goose/src/recipe/template_recipe.rs` +193/0,
  `documentation/docs/guides/recipes/recipe-reference.md` +21/−2;
  +348/−2

## Context

Recipe parameters today are `HashMap<String, String>`. To pass
structured context (multi-field signals, arrays of findings,
nested enrichment metadata) into a recipe's MiniJinja template,
callers had to flatten everything into per-field string keys and
lose the natural template shape (`{{ signal.namespace }}`,
`{% for item in findings %}`). MiniJinja itself supports dot-
notation, iteration, and conditionals on structured values
natively — the impedance mismatch was entirely on the goose-side
parameter type.

## Design

Two additive moves; no surface change to the existing string
path.

**Enum extension** (`recipe/mod.rs:+134`):
- `RecipeParameterInputType::Object` — "Structured object
  parameter passed as JSON. Enables `{{ param.field }}`."
- `RecipeParameterInputType::Array` — "Array passed as a JSON
  array. Enables `{% for item in param %}`."

Doc-comments on the variants name the template idiom they
unlock, which is the right shape for an enum that gets read
both by recipe authors (writing YAML) and by callers
(constructing parameters).

**Parallel render function** (`template_recipe.rs:+193`):
`render_recipe_content_with_structured_params` mirrors the
existing `render_recipe_content_with_params` exactly but
takes `HashMap<String, serde_json::Value>` instead of
`HashMap<String, String>`. The MiniJinja `Environment` setup
is byte-identical (same `UndefinedBehavior::Strict`, same
`recipe_dir`-scoped template loader, same security
posture). No new fs / eval / code-execution surfaces.

10 unit tests cover the load-bearing cases:
- String values backward compatibility (the new function
  still accepts string-shaped values)
- Object dot-notation, conditionals, 3-level nested access
- Array iteration, empty-array edge case
- Optional-field handling via `is defined` test
- Mixed scalar + object parameters in one render call
- Dot-access on a scalar value (expected error — must not
  silently coerce)
- Serde round-trip for both JSON and YAML
  deserialization of the new enum variants via
  `Recipe::from_content`

The PR is **disciplined about its own scope**: the
description explicitly defers (a) build-pipeline wiring
(`apply_values_to_parameters` and `build_recipe_from_template`
still operate on `HashMap<String, String>`), (b) the
`PUT /sessions/{id}/user_recipe_values` API surface, and
(c) desktop/CLI input widgets. That's the right call —
landing the template-rendering foundation without coupling
it to API contract changes lets the API discussion happen
on its own merits.

## Risks / nits

- The two new render entry points (`*_with_params` and
  `*_with_structured_params`) now diverge in signature
  but share an `Environment` setup and post-render
  pipeline. A future maintenance cost is that any
  security-relevant change (e.g. tightening
  `UndefinedBehavior`, adding a new template filter,
  changing the loader scope) must be applied to both. A
  shared private `build_environment(recipe_dir)` helper
  with the two public functions as thin wrappers that
  inject parameters would prevent accidental drift.
  Worth a follow-up.
- `Object` and `Array` variants serialize as
  `"object"` / `"array"` (snake_case via the existing
  serde rename pattern, presumably). The 3 round-trip
  tests confirm JSON and YAML both work; worth one
  explicit assertion that the on-disk YAML form is
  exactly `input_type: object` (not `Object` or
  `OBJECT`) so a recipe author can read the docs and
  type the right value.
- "Dot-access on scalar (expected error)" test —
  important to lock, because the failure mode of a
  template author writing `{{ scalar.field }}` and
  expecting empty-string-or-undefined rather than a
  hard error would be a footgun. Strict mode produces
  the right behavior here; the test pins it.
- The `documentation/docs/guides/recipes/recipe-
  reference.md +21/−2` update introduces the new
  variants but does not yet show the
  `apply_values_to_parameters` API limitation — a recipe
  author reading the docs will see "object/array
  supported" but if they then try to pass values through
  the existing build pipeline (deferred to a follow-up
  PR per the description) they'll hit a type error. A
  one-paragraph "currently you must call
  `render_recipe_content_with_structured_params`
  directly; full pipeline integration is a follow-up"
  caveat in the doc would set expectations correctly.

## Verdict

**`merge-after-nits`**

Right shape (parallel render function instead of
mutating the existing string path), right scope
(template foundation only, API/pipeline deferred),
right test coverage (10 cases including the
load-bearing scalar-error and serde round-trip).
Security posture is preserved (same
`UndefinedBehavior::Strict`, same template loader
scope, no new eval surface). Nits are
documentation-clarity (mention the deferred pipeline
wiring), one explicit on-disk YAML form assertion,
and a follow-up extraction of the shared `Environment`
setup to prevent drift between the two render entry
points.

## What I learned

The pattern of "add a parallel API instead of
generalizing the existing one" is the right move when
the existing API has many callers and the
generalization (here: `HashMap<String, Value>` instead
of `HashMap<String, String>`) would force every caller
to either change their type or `.into()`-coerce. The
parallel-API cost is a maintenance burden (two
functions to keep in sync); the
generalization-in-place cost is a callers'-type
migration. For a feature-add, parallel-API is the
lower-risk choice.
