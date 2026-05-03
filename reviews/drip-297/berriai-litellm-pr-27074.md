# Review: BerriAI/litellm PR #27074

- **Title:** fix(anthropic,bedrock,vertex): forward output_config.effort + 400 on garbage reasoning_effort
- **Author:** mateo-berri (Mateo Wang)
- **Head SHA:** `213c2ff6d7bb94b668528533a567401ba40cb289`
- **Verdict:** merge-after-nits

## Summary

Follow-up to #27039. Closes nine bugs surfaced by the QA matrix on that
PR. Three substantive changes:

1. Stop stripping `output_config.effort` on Bedrock Converse + Bedrock
   Invoke + Vertex AI Anthropic adaptive routes — previously every
   `low/medium/high/xhigh/max` collapsed to identical thinking on the
   wire because the helper popped `output_config` before signing.
2. Convert `ValueError` → `BadRequestError` (400 instead of 500) for
   garbage `reasoning_effort` and `output_config.effort` values.
3. Floor `reasoning_effort="minimal"` at the Anthropic provider
   minimum (1024 budget tokens) so it stops 400'ing on every direct
   Anthropic / Vertex / Bedrock Invoke call.

520 added / 113 deleted across 14 files; tests updated alongside.

## Specific-line comments

- `litellm/llms/anthropic/chat/transformation.py:822-832` — the
  `_map_reasoning_effort` minimal branch now uses
  `max(DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET, ANTHROPIC_MIN_THINKING_BUDGET_TOKENS)`.
  Correct; the constant is hard-pinned (not env-overridable per the
  comment in `constants.py`) which is the right call since 1024 is
  Anthropic's wire-protocol minimum, not a tunable.
- `litellm/llms/anthropic/chat/transformation.py:1099-1112` — wrapping
  `_map_reasoning_effort`'s `ValueError` into a `BadRequestError` with
  `llm_provider=self.custom_llm_provider or "anthropic"` is the right
  fallback. The fallback to `"anthropic"` is necessary because
  `custom_llm_provider` can be `None` on direct provider construction.
- `litellm/llms/anthropic/chat/transformation.py:1546-1602` — note the
  comment about `effort=""` previously being silently accepted via
  `if effort and ...`. Switching to `effort is not None and effort
  not in valid_efforts` is the right semantic. Empty string is a
  programmer error, not "no effort".
- `litellm/llms/bedrock/chat/converse_transformation.py:485-505` —
  the adaptive-thinking branch now stages `output_config.effort` on
  `optional_params` so the Anthropic-on-Bedrock path can pick it up
  in `_prepare_request_params`. Mapping `minimal -> low` (effort_map
  line 488) is reasonable for Claude 4.6/4.7 since adaptive thinking
  doesn't have a sub-`low` tier — but worth a one-line comment
  explaining why `minimal` collapses to `low` here while the non-
  adaptive path floors at 1024 tokens.
- `litellm/llms/bedrock/chat/converse_transformation.py:516-531`
  (`_supports_effort_level_on_bedrock`) — correctly looks up the
  bedrock-prefixed model entry directly in `litellm.model_cost`. The
  three-key probe (`model`, `base_model`, `f"bedrock/{base_model}"`)
  covers the common id shapes.
- `litellm/llms/bedrock/chat/converse_transformation.py:1306-1318` —
  `output_config` is popped via `anthropic_output_config = inference_params.pop(...)`
  but I don't see where that local variable is read in the diff
  excerpt. **Verify the rest of `_prepare_request_params` actually
  re-injects it into `additionalModelRequestFields` for `anthropic.*`
  models.** If the value is captured-but-unused, the silent-strip
  bug is still latent.

## Risks / nits

- The `llm_provider=self.custom_llm_provider or "anthropic"` fallback
  fires several times. Consider a small helper to compute it once.
- Tests cover the new `BadRequestError` paths and the `output_config`
  forwarding (1214 tests pass per PR description), but the QA matrix
  was 231 cells across 21 provider × model combos × 11 efforts —
  worth keeping that matrix as a smoke harness, not just one-off
  parametrize cases.
- Removing `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS` to an empty set is
  load-bearing on Vertex actually accepting `effort` on `:rawPredict`.
  PR description claims direct curls verify this on `claude-opus-4-7`
  global and `claude-opus-4-6` us-east5; trust + verify when this
  rolls out, and consider adding a recorded-cassette test.

## Verdict justification

Substantive bug-fix sweep with broad test coverage and a clear
narrative tying every change to a numbered QA finding. The one
concrete thing to verify before merge is the `anthropic_output_config`
re-injection point in `_prepare_request_params` — the diff shows the
pop but the truncation cuts off the use site. **merge-after-nits.**
