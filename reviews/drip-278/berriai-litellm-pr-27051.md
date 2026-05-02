# Review — BerriAI/litellm#27051

- PR: https://github.com/BerriAI/litellm/pull/27051
- Title: feat: map reasoning_effort=xhigh/max to budget_tokens on pre-4.6 Anthropic models
- Head SHA: `053f53fa89f9914f80fe4be07aaf382b5ff0e3b7`
- Size: +20 / −0 across 3 files
- Verdict: **merge-after-nits**

## Summary

Adds two new `reasoning_effort` levels — `xhigh` and `max` — to the
Anthropic chat transformation, mapping them to thinking-budget token
defaults of 8192 and 16384 respectively. Both are env-overridable via
`DEFAULT_REASONING_EFFORT_XHIGH_THINKING_BUDGET` and
`DEFAULT_REASONING_EFFORT_MAX_THINKING_BUDGET`.

## Evidence

- `litellm/constants.py:205-210` — two new module-level constants
  follow the existing `_HIGH_` pattern exactly:
  ```python
  DEFAULT_REASONING_EFFORT_XHIGH_THINKING_BUDGET = int(
      os.getenv("DEFAULT_REASONING_EFFORT_XHIGH_THINKING_BUDGET", 8192)
  )
  DEFAULT_REASONING_EFFORT_MAX_THINKING_BUDGET = int(
      os.getenv("DEFAULT_REASONING_EFFORT_MAX_THINKING_BUDGET", 16384)
  )
  ```
- `litellm/llms/anthropic/chat/transformation.py:14,17` — imports
  added in alphabetical order; clean.
- `litellm/llms/anthropic/chat/transformation.py:828-837` — two new
  `elif` branches inside `_map_reasoning_effort` for `"xhigh"` and
  `"max"`, both producing `AnthropicThinkingParam(type="enabled",
  budget_tokens=…)`. Falls through to the existing
  `raise ValueError(f"Unmapped reasoning effort: {reasoning_effort}")`
  for unknown values, preserving current contract.
- `tests/test_litellm/llms/anthropic/chat/test_anthropic_chat_transformation.py`
  — parametrize list extended with `("xhigh", 8192)` and
  `("max", 16384)`, so the existing `_map_reasoning_effort` test
  matrix now covers both.

## Notes / nits

- Title says "pre-4.6 Anthropic models" but the implementation does
  *not* gate on model version anywhere — `_map_reasoning_effort`
  is a pure transformation. If 4.6+ models accept the same
  `thinking.budget_tokens` shape, this is fine and the title is
  imprecise; if 4.6+ expect a different shape, this PR will
  misroute requests for them. Please clarify in the PR body or add
  a model-version guard.
- `xhigh` and `max` are not standard OpenAI `reasoning_effort` values.
  Worth a one-line note in the relevant proxy docs that these are
  litellm-specific extensions, otherwise users hitting a non-Anthropic
  provider with `xhigh` will get a `ValueError` from the corresponding
  provider mapper (or worse, silent acceptance with no thinking
  budget).
- The 8192 / 16384 defaults are reasonable but undocumented in the
  diff. A short comment next to each constant (or a row in the
  reasoning-effort docs table) would help operators sizing latency
  budgets.

Mechanically clean. Ship after the title/version-gate clarification.
