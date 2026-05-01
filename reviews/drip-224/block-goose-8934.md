# block/goose #8934 — feat(recipe): support structured parameters (object/array) in recipe templates

- **Repo:** block/goose
- **PR:** https://github.com/block/goose/pull/8934
- **HEAD SHA:** `826a5cf11fcb10510458352c35402df99e4877d6`
- **Author:** jordigilh
- **Verdict:** `merge-after-nits`

## What the diff does

Multi-file additive feature adding `Object` and `Array` variants to
`RecipeParameterInputType`, threading structured (JSON) parameter values
end-to-end through the recipe build pipeline so MiniJinja templates can
use dot-notation (`{{ signal.namespace }}`), iteration
(`{% for item in findings %}`), and conditionals on structured data:

1. **Enum extension.** `crates/goose/src/recipe/mod.rs` adds `Object` and
   `Array` to `RecipeParameterInputType`.

2. **New rendering function.**
   `crates/goose/src/recipe/template_recipe.rs` adds
   `render_recipe_content_with_structured_params` accepting
   `HashMap<String, serde_json::Value>` (vs the existing
   `HashMap<String, String>`), reusing the same MiniJinja `Environment`
   setup (Strict undefined behavior, recipe-dir-scoped template loader).

3. **Pipeline routing.** `crates/goose/src/recipe/build_recipe/mod.rs:38-65`
   detects `has_structured_params` via
   `recipe_parameters.is_some_and(|params| params.iter().any(|p|
   matches!(p.input_type, Object | Array)))` and routes through the new
   structured renderer; otherwise the existing string path is used (zero
   overhead for the pure-string case).

4. **`to_structured_params` helper** at `build_recipe/mod.rs:69-122` —
   converts the string map to a `HashMap<String, Value>`, parsing JSON
   only for keys whose `input_type` is Object/Array; other keys remain
   `Value::String`. Critically, the helper validates at conversion time
   (commit `83db5311`) that the parsed `Value` matches the declared
   `input_type` — Object → `is_object()`, Array → `is_array()` — and
   raises a typed error naming both the expected and received shape via
   the new `json_type_name` helper at `:127-135`.

5. **Validation-mode flip** at `parse_recipe_content` (commit `f174a245`)
   — switches the discovery-pass MiniJinja `Environment` from
   `UndefinedBehavior::Lenient` to `UndefinedBehavior::Chainable` because
   Lenient forbids attribute access on undefined values
   (`{{ signal.name }}` fails the discovery render even though the
   parameter is undefined-by-design in the discovery pass), while
   Chainable returns undefined and continues. The actual rendering phase
   keeps Strict.

6. **Bracket notation handling** (commit `3e953636`) —
   `extract_root_identifier` handles both `signal.name` and
   `findings[0].name` / `signal["ns"]` shapes when normalizing template
   variables to parameter keys, with a follow-up `clippy::string_slicing`
   fix at commit `826a5cf1` replacing `&var[..pos]` with `str::split` on
   `.`/`[` delimiters (UTF-8 safe).

7. **Generated-types update.** `openapi.json`, `types.gen.ts`, and the
   desktop `recipeFormSchema.ts` Zod enum updated (commits `8d152fda`,
   `7626d92f`).

8. **Tests** — 18 new tests across three files: 10 unit tests in
   `template_recipe.rs` (string compat, object dot-notation, conditionals,
   array iteration, nesting, optional fields, mixed params, edge cases),
   4 integration tests through `build_recipe_from_template` in
   `build_recipe/tests.rs` (object param, array param, mixed
   string+object, invalid JSON error, type-mismatch error), and 4 serde
   round-trip tests in `mod.rs`.

## Why the change is right

The shape is **opt-in additive feature with the right zero-overhead
fallback**: the structured path only fires when at least one parameter
declares `input_type: object | array`, otherwise the existing
`render_recipe_content_with_params` string path runs unchanged.
Backwards compat is preserved because `Vec<(String, String)>` is still
the API surface on `build_recipe_from_template` and all callers
(`goose-cli`, `goose-server`, `summon`, `execute_commands`) — JSON-encoded
strings flow in, the helper does the typed parse internally.

The validation-at-conversion-time at `to_structured_params:97-117` is the
load-bearing security/UX detail: rather than letting a string-shaped
value flow into MiniJinja and fail with a confusing "object has no
attribute 'name'" error, the helper rejects with
`"Parameter 'signal' has input_type object but received an array"` at the
boundary. The `json_type_name` helper at `:127-135` is the right
human-readable formatting layer ("a string", "an array", etc.) for the
error message.

The `Lenient → Chainable` flip in `parse_recipe_content` (commit
`f174a245`) is the right fix for the discovery-pass-vs-render-pass
distinction: the discovery pass exists to enumerate template variables
without parameter values; Strict/Lenient both fail on undefined
attribute access, but Chainable returns undefined-up-the-chain so the
discovery walk completes. The render pass keeps Strict for safety. This
correctly preserves the safety invariant where it matters (real
rendering) while loosening it only where the design demands it
(discovery walk over an explicitly-empty variable space).

The bracket-notation handling at `extract_root_identifier` (commit
`3e953636`) closes the obvious follow-on gap: a template using
`findings[0].name` should still resolve to the `findings` parameter for
the validate-parameters-in-template pass. The clippy follow-up at
`826a5cf1` fixing the byte-index slicing is the right
multi-byte-UTF-8-safe replacement.

The security-model statement in the PR body is correct: same MiniJinja
environment setup, same Strict undefined behavior, same recipe-dir-scoped
loader, no new filesystem access or eval surface. The structured
parameter values are rendered by MiniJinja's value model, which has no
code-execution affordance regardless of whether values are strings or
objects.

## Nits (non-blocking)

1. **Many "Made-with: Cursor" trailers in commits.** Cosmetic — most
   projects squash trailers like this on merge, but worth flagging that
   the commit history has 13 such trailers. If maintainers prefer
   trailer-free history, a squash-merge or interactive cleanup would
   help. Not a code issue.

2. **Out-of-scope acknowledgment is correct but worth a follow-up
   tracking issue.** PR body lists desktop/CLI UX (JSON-aware input
   widgets) and API surface (`PUT /sessions/{id}/user_recipe_values`
   still `HashMap<String, String>`) as deferred. A linked tracking
   issue would prevent these from getting lost — without UX support,
   end users have no obvious way to *enter* JSON for structured params
   from the recipe editor.

3. **`render_recipe_content_with_structured_params` test for
   recipe-dir-scoped `extends`/`include`.** PR body states the new
   function uses "the same recipe_dir-scoped template loader for
   extends/include," but the 10 unit tests don't include an
   extends/include arm. A single test with a `{% include "child.yaml" %}`
   that itself uses `{{ signal.name }}` would pin the contract that the
   loader is wired correctly into the structured path.

4. **`to_structured_params` ordering of validation vs parse.** At
   `:91-96` the helper does `serde_json::from_str` first and then
   checks `is_object()`/`is_array()`. For a string value
   `"\"hello\""` declared as `input_type: object`, the parse succeeds
   (as a JSON string) and the type-check then rejects with "received a
   string." This is correct behavior, but the error message could
   confuse a user who passed `{"hello": "world"}` and got "received a
   string" because of mis-quoted shell escaping. Worth a
   "Hint: shell-escape your JSON" addendum on the error.

5. **`json_type_name` lives in `build_recipe/mod.rs` rather than a
   shared utility.** It's a 9-line helper and unlikely to be needed
   elsewhere, but if a future caller of the structured-params helper
   needs the same human-readable type name, dragging this out into a
   shared `recipe::json_utils` would prevent duplication.

## Verdict rationale

Right-shaped additive feature with correct opt-in routing, zero-overhead
string fallback, validation-at-conversion-time at the typed-error
boundary, and a substantial 18-test coverage matrix. The discovery-pass
Lenient→Chainable flip is the right fix for the right distinction, and
the bracket-notation extension closes the obvious follow-on gap. Nits
are documentation/follow-up tracking issues, not blockers, but the
deferred UX/API-surface work should be tracked so structured params
don't ship without an obvious way for end-users to provide them from the
desktop/CLI flows.

`merge-after-nits`
