# Review: BerriAI/litellm #27089 — Native JSON mode for Claude 4.7 + max_completion_tokens

- PR: https://github.com/BerriAI/litellm/pull/27089
- Head SHA: `8f60e503c1363f9d67f5719bd8634e3dd0caec22`
- Author: koushik1124
- Files touched: `litellm/llms/anthropic/chat/transformation.py`, `tests/test_litellm/test_parity_gap.py` (new)

## Verdict: `merge-after-nits`

## Rationale

The patch extends the alias allow-list at `litellm/llms/anthropic/chat/transformation.py:1054-1063` so that `sonnet-4.7`, `sonnet-4-7`, `sonnet_4.7`, `sonnet_4_7`, `4.7-sonnet`, `4-7-sonnet`, and `4_7_sonnet` route to the native `output_format` JSON path rather than falling back to tool-calling. This is the same pattern already used for 4.5 / 4.6 a few lines up, so consistency is good and the test at `tests/test_litellm/test_parity_gap.py:5-31` asserts the mapping for a versioned model id (`claude-4-7-sonnet-20261031`). The `max_completion_tokens` tests for Anthropic and Bedrock at lines 33-71 don't exercise any new code in this diff — they're validation tests of pre-existing behavior; harmless but slightly off-topic for the PR title. Two nits: (1) the alias matcher is pure substring containment (`any(... in model)`), so `claude-3.7-sonnet` will *not* match `sonnet-4.7` but `sonnet-4.7-experimental-foo` would — that's the intended fuzzy behavior, but if any past model carried "4-7" in an unrelated position (e.g. a date `2024-04-07`) it would falsely activate; a regex anchored to non-digit boundaries would be safer long-term. (2) The new test file is named `test_parity_gap.py` with no docstring at module level explaining the scope — a one-liner saying "regression coverage for openai-compat parity on response_format/max_completion_tokens across providers" would help future maintainers. Merge after either tightening the matcher or at least leaving a TODO comment.