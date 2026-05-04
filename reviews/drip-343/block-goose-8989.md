# block/goose #8989 — fix(extension-manager): require extension_name on read_resource

- PR: https://github.com/block/goose/pull/8989
- Head SHA: `6aab98f2ed7d2bac6c323002844fdd88e5a73528`
- Diff size: +26 / -79 across 2 files

## Summary

Tightens the `read_resource` MCP tool to **require** an
`extension_name` parameter instead of treating it as optional. The
old implementation, when no extension name was provided, would loop
through every extension that supports resources and try them
sequentially, returning the first match. This PR removes the search
fallback entirely. Also fixes a bug where a sibling tool was looking
up `params.get("extension")` instead of `params.get("extension_name")`.

## Citations

- `crates/goose/src/agents/extension_manager.rs:1306-1320` — the
  function now `require_str_parameter`s `extension_name`. The whole
  search-loop branch (~50 lines) is deleted. Net behavior change:
  previously-permissive callers that relied on the search fallback
  will start getting parameter-validation errors.
- `crates/goose/src/agents/extension_manager.rs:1439` — fixes
  `params.get("extension")` → `params.get("extension_name")` in
  what looks like a parallel resource-listing handler. This is a
  *latent* bug fix piggybacking on the contract tightening; worth
  calling out separately in the commit message because it's a
  behavior change of its own (the previous code was silently always
  hitting the `None` branch since the param name didn't exist).
- `crates/goose/src/agents/platform_extensions/ext_manager.rs:54-56`
  — `ReadResourceParams.extension_name` changed from
  `Option<String>` (with `skip_serializing_if`) to `String`. This
  is a **breaking JSON-schema change** for any client that was
  omitting the field. Before merge, confirm:
    1. The tool description (text shown to the LLM) is updated to
       reflect the now-required field — the diff's docstring at
       `:325-328` still talks about "If no extension is provided,
       the tool will search all extensions for the resource",
       which is now a lie.
    2. Migration story for existing sessions / saved tool calls
       that have `extension_name: null` in their replay logs.
- The remaining hunks at `:329-340` are pure formatting churn
  (rustfmt re-flow of a chained `.expect().clone()` call). Harmless
  but adds review noise.

## Risk

Moderate. This is an MCP tool contract change. LLMs that learned the
old "extension_name is optional" shape from prior tool descriptions
will now fail validation until prompts are refreshed. The deleted
search logic was *useful* at the user-experience level — agents
could ask for a resource by URI without knowing which extension
exposed it. Forcing the agent to know the extension name shifts
discovery burden onto the model.

## Verdict

`needs-discussion` — the underlying simplification is reasonable
(unique-resolution-per-URI is a stronger contract) and the
`extension` → `extension_name` typo fix is unambiguously good. But
the docstring still describes the old behavior, and the search
fallback removal is a UX regression that deserves a paired update to
the tool description and probably a migration note. Should not merge
until docstring and tool description are aligned with the new
contract.
