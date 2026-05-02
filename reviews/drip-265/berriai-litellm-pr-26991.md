# BerriAI/litellm #26991 — fix(bedrock): forward output_config.effort for adaptive-thinking Claude

- **Repo:** BerriAI/litellm
- **PR:** #26991
- **Head SHA:** `f52cbae60c259807e21af19f28b99743780b4f6c`
- **Verdict:** merge-after-nits

## Summary

Bedrock was unconditionally stripping `output_config` on every Claude
route, which is correct for pre-4.6 Claude (those reject the field)
but silently dropped `effort` for Sonnet 4.6 / 4.7 / Opus 4.5. PR
splits the strip behavior by model:

- **Bedrock Invoke `/v1/messages`:** new `_supports_effort_on_bedrock`
  in `bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py`
  preserves `output_config` for adaptive-thinking + Opus 4.5.
- **Bedrock Invoke `/completion`:** symmetric `_supports_effort_on_bedrock_invoke`
  in `bedrock/chat/invoke_transformations/anthropic_claude3_transformation.py:35-46`.
- **Bedrock Converse:** new `_fold_output_config_effort_into_thinking`
  in `bedrock/chat/converse_transformation.py:478-518` translates
  `output_config.effort` → `thinking.effort` (Converse takes effort
  via `additionalModelRequestFields.thinking.effort`, not as a
  top-level block).
- Beta header config: `effort-2025-11-24` no longer maps to `null`
  for `bedrock` (`anthropic_beta_headers_config.json:106`), so the
  auto-attach for Opus 4.5 actually fires.

## Specific notes

- **Nit (`converse_transformation.py:455-475`):** the inline
  `effort_map = {"low": "low", "minimal": "low", ...}` duplicates
  what `_map_reasoning_effort` already knows. Consider exposing a
  single canonical mapper on `AnthropicConfig` instead. Not blocking.
- **Concern (`anthropic_claude3_transformation.py:35-46`):** the
  Opus-4.5 detection uses `any(p in model_lower for p in ("opus-4.5",
  "opus_4.5", "opus-4-5", "opus_4_5"))`. Substring matching invites
  drift; `claude-opus-4.5-pro-max` style future SKUs are fine, but a
  hypothetical `opus-4-5-experimental-bad-build` would also match.
  Consider centralizing in `AnthropicModelInfo` next to
  `_is_adaptive_thinking_model`.
- **Behavior preservation:** the converse path drops the second
  `inference_params.pop("output_config", None)` (lines ~1280-1283
  in old vs new) because the fold helper has already consumed it
  and the earlier pop at the top covers the strip case. Verified —
  no leak to the wire body.
- **Tests:** the listed test functions cover Sonnet 4.6, Opus 4.6,
  Opus 4.7 (regional + global), Opus 4.5 + beta, and the
  fold-preserves-existing-thinking precedence rule. Good coverage.

## Rationale

Real customer-facing bug, well-scoped fix that respects the
historical reason for the unconditional strip. Two nits about model
detection centralization — neither blocks merge.
