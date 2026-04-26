# PR #26538 — fix(fireworks_ai): modernize chat transforms, add Messages + Responses configs

- **Repo**: BerriAI/litellm
- **PR**: #26538
- **Head SHA**: `88fe6c9ccfb14d90e57558ae107528e81c0463ad`
- **Author**: frdeng
- **Size**: +1005 / -172 across 12 files (with substantial new test coverage)
- **Verdict**: **merge-after-nits**

## Summary

Sweeping refresh of the Fireworks AI provider integration. (1)
Restructures the chat-transform parameter logic, (2) adds new
`FireworksAIMessagesConfig` and `FireworksAIResponsesConfig`
classes for the Messages and Responses API endpoints, (3)
registers both in `litellm/__init__.py` and the lazy-imports
registry, (4) lifts the docstring into a structured block
documenting Fireworks-specific transforms.

## Specific changes

- `litellm/llms/fireworks_ai/chat/transformation.py:42-89` —
  rewritten class docstring documenting four buckets: request
  transforms (model name prefixing, `#transform=inline` for
  document inlining, file→image migration, field stripping of
  `cache_control` + `provider_specific_fields`), response
  transforms (model prefixing, tool-calls-in-content workaround
  for Llama-v3p3-70b), parameter handling, and capability
  flag wiring. This is real documentation, not aspirational.
- `chat/transformation.py:91-93` — adds
  `custom_llm_provider` property returning `"fireworks_ai"`.
  Standard pattern in the rest of the codebase.
- `chat/transformation.py:138-167` — `get_supported_openai_params`
  now unconditionally includes `top_logphobs`, `seed`,
  `logit_bias`, `parallel_tool_calls`, `thinking`,
  `reasoning_effort`. The old code gated `reasoning_effort`
  behind `supports_reasoning(...)`; the new docstring at L86-88
  justifies dropping that gate ("Fireworks API accepts it on all
  models and handles unsupported cases itself"). **This is the
  one nit** — silently passing an unsupported param to the
  upstream and trusting them to ignore it is a behavior change;
  worth a regression test for the model that previously *omitted*
  `reasoning_effort` from outbound requests.
- `chat/transformation.py:178-195` — `map_openai_params`
  simplified: previously had a special-case for
  `tool_choice == "required"` → `"any"` (referencing
  https://github.com/BerriAI/litellm/issues/4416), now
  passes value through unchanged. **This is a semantic
  regression risk** — confirm Fireworks now accepts `"required"`
  natively, otherwise users on that flow will start getting
  4xx responses.
- `litellm/__init__.py:1798-1803` and
  `_lazy_imports_registry.py:1014-1021` — registers two new
  configs. Lazy registry entries match the eager imports.
- New `llms/fireworks_ai/messages/transformation.py` (+62) and
  `llms/fireworks_ai/responses/transformation.py` (+54). Not
  visible in head-200 but the file sizes and test sizes
  (199 + 147 lines respectively) suggest meaningful new logic.
- Tests: `test_fireworks_ai_chat_transformation.py` +265 / -29,
  `test_fireworks_ai_messages_transformation.py` (+199 new),
  `test_fireworks_ai_responses_transformation.py` (+147 new),
  `test_fireworks_ai_cost_calculator.py` +84. Solid coverage.

## Risks

1. **`tool_choice="required"` regression** (above). High-priority
   nit — needs a test that asserts the outbound payload still
   gets `"required"` translated to whatever Fireworks expects, or
   a clear note that Fireworks now accepts `"required"` natively.
2. **Always-pass `reasoning_effort`** could leak the param to
   non-reasoning models that previously ignored it cleanly but
   might 4xx now. Test against at least one non-reasoning Fireworks
   model.
3. The `cost_calculator.py` change is +19/-17 — invisible from
   the head-200 cut but pricing changes deserve their own
   reviewer attention.

## Verdict

`merge-after-nits` — the structural improvements (separate
Messages/Responses configs, structured docstring) are clear
wins. Two semantic changes (tool_choice="required" passthrough,
unconditional reasoning_effort) need verification before merge.

## What I learned

The pattern "remove gate X because the upstream now handles
unsupported cases" is a specific kind of risk: it works *until*
the upstream changes its tolerance. A defensive position is to
keep the gate but log a warning when it would have dropped the
param — gives you the same behavior with an audit trail.
