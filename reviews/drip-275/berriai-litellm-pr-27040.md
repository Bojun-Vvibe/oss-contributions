# Review: BerriAI/litellm #27040

- **PR:** BerriAI/litellm#27040 — `fix(bedrock): forward output_config.effort for adaptive-thinking Claude`
- **Head SHA:** `2a2fcb32a63b97ccb812f1cd1770fdfe09cf1e94`
- **Files:** 8 (+354/-8)
- **Verdict:** `merge-after-nits`

## Observations

1. **Three-route split is correct** — The PR description correctly identifies that Bedrock exposes `effort` differently across (a) Invoke `/v1/messages` (top-level `output_config.effort`), (b) Invoke `/completion` (top-level `output_config.effort`), and (c) Converse (nested `additionalModelRequestFields.thinking.effort`). The fix touches all three transformation modules and adds a model-aware gate (`AnthropicModelInfo._is_adaptive_thinking_model(model) or _is_claude_opus_4_5(model)`) in each. This avoids regressing pre-4.6 Claude that 400s on `output_config`.

2. **Converse fold logic at `converse_transformation.py:471-516` is the cleverest piece** — `_fold_output_config_effort_into_thinking` translates the Anthropic Messages-shaped `output_config.effort` into the Converse-shaped `thinking.effort` only when (a) the model is adaptive-thinking, (b) `output_config.effort` is a non-empty string, and (c) `thinking.effort` isn't already set by the caller. The "don't override explicit caller value" branch is exactly right. One subtle thing: when `thinking` exists but its `type != "adaptive"` (e.g. caller passed `enabled` for legacy budget mode), the helper still writes `thinking["effort"] = effort`, which is rejected by the AWS API for non-adaptive thinking. Consider gating the write on `thinking.get("type") == "adaptive"`.

3. **Beta-headers config (`anthropic_beta_headers_config.json`)** flips `"effort-2025-11-24": null → "effort-2025-11-24"`. Mapping a beta to itself is the conventional way to opt a provider into auto-attach; the PR description correctly explains this is needed for Opus 4.5 since the auto-attach path keys off `is_effort_used`. Worth a one-line code comment in the JSON or adjacent loader so future readers don't think this is a copy-paste typo.

4. **`reasoning_effort` mapping in `_handle_reasoning_effort_parameter` (lines 452-475)** uses a hardcoded `effort_map` with `xhigh` and `max` — these aren't in the public OpenAI `reasoning_effort` enum (`minimal/low/medium/high`). The fallback `effort_map.get(reasoning_effort, reasoning_effort)` will pass through whatever the caller sent, which is fine, but the explicit `xhigh`/`max` entries imply the codebase has internal callers using those values; a comment pointing to where they're produced would help.

## Recommendation

Strong fix with thorough test coverage. Address the `thinking.type != "adaptive"` edge case before merge.
