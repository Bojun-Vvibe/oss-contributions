# BerriAI/litellm PR #27053 — fix(anthropic,vertex): three reasoning_effort bugs surfaced by PR #27039 QA matrix

- **Repo:** BerriAI/litellm
- **PR:** #27053
- **Head SHA:** `ca5c24c56edc68f1c831d868c91a76f2899f9192`
- **Author:** mateo-berri
- **Title:** fix(anthropic,vertex): three reasoning_effort bugs surfaced by PR #27039 QA matrix
- **Diff size:** +175 / -52 across 4 files
- **Drip:** drip-293

## Files changed

- `litellm/llms/anthropic/chat/transformation.py` (+~36/-~3) — drops the `DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET` import; introduces module-level `_VALID_REASONING_EFFORT_VALUES = frozenset({"none","minimal","low","medium","high","xhigh","max"})` (with extensive justification comment); adds an up-front validation branch in `_map_reasoning_effort` (lines ~820-829) raising `ValueError` for unrecognized strings *before* the 4.6/4.7 adaptive short-circuit; reroutes `minimal` to use `DEFAULT_REASONING_EFFORT_LOW_THINKING_BUDGET` (1024) instead of `DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET` (128).
- `litellm/llms/vertex_ai/.../experimental_pass_through/transformation.py` (~5 line comment update) — replaces the "Vertex rejects effort" comment with a corrected one referencing direct `:rawPredict` curl verification.
- `litellm/llms/vertex_ai/.../output_params_utils.py` (~17 line comment + behavior change) — empties `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS` from `frozenset({"effort"})` to `frozenset()`. Keeps the sanitize scaffolding for future use with updated docstring.
- `litellm/llms/vertex_ai/.../anthropic/transformation.py` (~5 line comment update) — mirror of the experimental_pass_through comment fix.
- `tests/test_litellm/llms/anthropic/chat/test_anthropic_chat_transformation.py` (+~58/-1) — updates the `("minimal", 128)` parametrize case to `("minimal", 1024)`; adds `test_minimal_reasoning_effort_emits_at_least_anthropic_min_budget` asserting `>= 1024` across haiku-4-5 / sonnet-4-5 / opus-4-5; adds `test_invalid_reasoning_effort_is_rejected_uniformly` parametrized across 4.6/4.7 + pre-4.6 models × `["disabled","invalid","","garbage"]`.

## Specific observations

- `transformation.py:94-110` — module-scope `_VALID_REASONING_EFFORT_VALUES` is correctly placed *outside* `AnthropicConfig` to avoid leaking through `BaseConfig.get_config`'s `cls.__dict__` walk. The justification comment is exactly the kind of thing future contributors need; well done. The leading underscore signals private-to-module, which is right.
- `transformation.py:820-829` — placing the validation branch *before* the 4.6/4.7 short-circuit is the correct fix for QA bug #3 (silent `adaptive` mapping for invalid values). Raising `ValueError` is consistent with the existing `else: raise ValueError(...)` at the bottom of the function. The comment explicitly defers the `ValueError → litellm.BadRequestError` upgrade to PR #27050 to avoid textual conflict — that's mature merge hygiene.
- `transformation.py:855-870` — rerouting `minimal` to `DEFAULT_REASONING_EFFORT_LOW_THINKING_BUDGET` (1024) rather than the 128 fallback is correct for clearing the Anthropic API minimum. The comment naming all 4 affected providers (Anthropic direct / Azure AI Anthropic / Bedrock Invoke / Vertex AI Anthropic) and noting Bedrock Converse silently clamped is exactly the diagnostic context to preserve. One nit: the comment names `DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET_ANTHROPIC` in a test edit (line ~2161) but the actual constant introduced/used here is `DEFAULT_REASONING_EFFORT_LOW_THINKING_BUDGET`. Update the test parametrize comment to match the constant the code actually uses.
- `output_params_utils.py:13-30` — emptying `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS` is correct *given* the curl verification, but it's a server-behavior assertion. Worth leaving a TODO for someone to re-test in 90 days, since Vertex parity has historically drifted (the comment even acknowledges this). The sanitize scaffolding is correctly kept.
- `output_params_utils.py:23` — frozen comment "Direct ``:rawPredict`` curls against ``us-east5`` on opus-4-7 ... and opus-4-6/sonnet-4-6" — pin the curl date or commit SHA in the comment so the next maintainer knows when to re-verify.
- Tests are dense and well-targeted: parametrizing across both 4.6/4.7 (which previously short-circuited) *and* pre-4.6 (which already raised) captures the regression matrix exactly. Good.
- Concern: emptying `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS` removes the *only* runtime gate. If Vertex regresses, the failure mode is now a 400 in production. Consider keeping a feature-flag or env-var override for re-enabling the strip if customers report breakage.

## Verdict: `merge-after-nits`

This is a well-engineered three-bug fix with strong test coverage and exemplary inline justification. Three nits: (1) align the test comment to reference the actual constant `DEFAULT_REASONING_EFFORT_LOW_THINKING_BUDGET` rather than the non-existent `_MINIMAL_THINKING_BUDGET_ANTHROPIC`, (2) pin the curl verification date in the Vertex comment, (3) consider an env-var rollback knob for the Vertex sanitize change. Otherwise textbook PR.
