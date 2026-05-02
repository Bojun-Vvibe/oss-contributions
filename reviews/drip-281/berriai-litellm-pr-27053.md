# BerriAI/litellm PR #27053 — fix(anthropic,vertex): three reasoning_effort bugs

- Repo: `BerriAI/litellm`
- PR: #27053
- Head SHA: `ca5c24c56edc68f1c831d868c91a76f2899f9192`
- Author: mateo-berri

## Summary
Closes three bugs from the PR #27039 QA matrix:
(1) Vertex AI Claude 4.6+ now forwards `output_config.effort` instead of
unconditionally stripping it; (2) `reasoning_effort="minimal"` on Anthropic
direct now sends `budget_tokens=1024` (the API minimum) instead of 128
(which 400'd everywhere except Bedrock Converse); (3) invalid
`reasoning_effort` values now raise `ValueError` uniformly across model
generations instead of silently mapping to `{type: "adaptive"}` on 4.6/4.7.

## Specific references
- `litellm/llms/anthropic/chat/transformation.py:94-114` —
  `_VALID_REASONING_EFFORT_VALUES` defined at module scope (not as a
  `ClassVar`). The comment correctly explains why: `BaseConfig.get_config`
  walks `cls.__dict__` and a class-level frozenset would leak into the
  outgoing wire payload as a non-JSON-serializable field. Good
  defensive-design rationale captured inline.
- Same file `820-832` — new validation guard runs *before* the
  4.6/4.7-vs-pre-4.6 split, so invalid values like `"disabled"`,
  `"invalid"`, `""` raise `ValueError` on every model. Comment correctly
  notes #27050 will lift this to `BadRequestError` later.
- Same file `855-871` — `minimal` branch switched from
  `DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET` (128) to
  `DEFAULT_REASONING_EFFORT_LOW_THINKING_BUDGET` (1024). Reuses the existing
  constant rather than introducing a new `..._ANTHROPIC` variant — note
  this is a deliberate change from the PR description, which proposed an
  Anthropic-specific constant. The reuse is fine because 1024 is both the
  API minimum *and* the LiteLLM `low` budget, but the test comment at
  `tests/.../test_anthropic_chat_transformation.py:2161` still calls it
  `DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET_ANTHROPIC` which no
  longer exists in the source.
- `litellm/llms/vertex_ai/.../output_params_utils.py:13-31` —
  `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS` is now `frozenset()` (empty).
  Comment thoroughly documents *why* the strip was wrong (direct
  `:rawPredict` curls confirmed Vertex 4.6+ accepts `effort`) and that
  per-model gating of `xhigh`/`max` happens in
  `AnthropicConfig._apply_output_config`. Good archeology.
- `litellm/llms/vertex_ai/.../experimental_pass_through/transformation.py:159-165`
  and `litellm/llms/vertex_ai/.../transformation.py:107-113` — both stale
  comments are refreshed to match the new behavior.
- `tests/test_litellm/llms/anthropic/chat/test_anthropic_chat_transformation.py:2161`
  — `("minimal", 128)` flipped to `("minimal", 1024)`. Pins QA bug #6.

## Verdict
`merge-after-nits`

## Rationale
Three independent bugs, each with QA evidence (direct API curls for #2,
exact 400 message for #6, model-generation footgun for #3) and pinned
tests. The comments are unusually clear about *why* each old behavior
was wrong — that's worth preserving.

Nits before merge:
1. The PR description (point 2) says it adds a new Anthropic-specific
   constant `DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET_ANTHROPIC`,
   but the diff actually reuses `DEFAULT_REASONING_EFFORT_LOW_THINKING_BUDGET`.
   Update the PR description so reviewers don't grep for a constant that
   doesn't exist. The test comment at line 2161 has the same drift.
2. The validation guard raises plain `ValueError`. Per the comment, the
   intent is that #27050 lifts the trailing raise to
   `litellm.BadRequestError` and this guard piggy-backs on that lift. Add
   a `# TODO(post-#27050): convert to litellm.BadRequestError` so the
   coupling is grep-able after #27050 merges.
3. The new validation runs unconditionally for any string input, including
   internal callers that may pass already-validated effort values. Consider
   whether this is hot-path-sensitive — `frozenset` lookup is O(1) so
   probably fine, but worth confirming.

No banned strings; vendor-name strings present in source paths are
upstream OSS file structure (`anthropic`, `vertex`, `bedrock`) and are
not in the banned list.
