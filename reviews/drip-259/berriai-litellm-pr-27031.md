# BerriAI/litellm PR #27031 — [Test] Anthropic: Replace Legacy Claude-4-Sonnet Alias With Haiku 4.5

- PR: https://github.com/BerriAI/litellm/pull/27031
- Head SHA: `abfaab5dc348b957c5ff0f5938e1effaf2abc8ca`
- Author: @yuneng-berri (yuneng-jiang)
- Size: +7 / -7

## Summary

Three live-API tests pinned to `claude-4-sonnet-20250514` (a non-canonical alias of `claude-sonnet-4-20250514`) fail with `not_found_error` against freshly issued Anthropic keys because the legacy alias no longer resolves. A fourth test pinned to the canonical `claude-sonnet-4-20250514` snapshot is also two weeks from its 2026-05-14 deprecation. PR bumps all four sites — across `tests/local_testing/test_streaming.py`, `tests/local_testing/test_function_calling.py`, `tests/litellm_utils_tests/test_anthropic_token_counter.py`, and the two `tests/pass_through_tests/` files — to `claude-haiku-4-5-20251001`.

The capability set required (streaming, parallel tool calling, extended thinking, token counting) is a strict subset of Haiku 4.5's, and the model is cheaper per-token. PR is test-only, no library code changes.

## Specific references from the diff

- `tests/litellm_utils_tests/test_anthropic_token_counter.py:29` — `get_test_model` returns `claude-haiku-4-5-20251001`.
- `tests/local_testing/test_function_calling.py:161` — parametrize row for `test_aaparallel_function_call_with_anthropic_thinking` swapped to `anthropic/claude-haiku-4-5-20251001`. The bedrock parametrize peer (`bedrock/us.anthropic.claude-sonnet-4-5-20250929-v1:0`) intentionally untouched.
- `tests/local_testing/test_streaming.py:1273` and `:2699` — parametrize row + `test_completion_claude_3_function_call_with_streaming` model arg.
- `tests/pass_through_tests/base_anthropic_messages_test.py:57, :78` — both `test_anthropic_messages_with_thinking` and `test_anthropic_streaming_with_thinking` now use Haiku 4.5 for the `thinking={"type": "enabled", "budget_tokens": 16000}` cases.
- `tests/pass_through_tests/test_anthropic_passthrough.py:336` — `test_anthropic_messages_streaming_cost_injection` payload model.

## Verdict: `merge-after-nits`

Right fix, well-scoped, no library impact, the rationale on capability-superset and deprecation timeline is sound. Two things keep this off `merge-as-is`.

## Nits / concerns

1. **Cost-injection test now exercises a different price tier.** `test_anthropic_messages_streaming_cost_injection` at line 336 verifies that cost is correctly attached to streaming responses. Sonnet 4 was \$3/\$15 per Mtok; Haiku 4.5 is \$1/\$5. If the test asserts a *specific* cost value (vs. just "cost > 0 and present"), the assertion will need updating. Worth a quick scan of the assertion body — the diff only shows the model swap, not the assert lines.
2. **PR description acknowledges ~11 other `claude-4-sonnet-20250514` references** in non-live unit-test fixtures. Leaving them out is the right call for this PR (changing them requires re-baselining cost-calc expected values), but please file a follow-up issue tagged `tech-debt` so they get migrated before the canonical `claude-sonnet-4-20250514` snapshot deprecates on 2026-05-14. Otherwise some of those fixtures will become unreachable in CI without anyone noticing until the next time someone touches them.
3. **The thinking-with-streaming pair (`base_anthropic_messages_test.py:57, :78`) sets `budget_tokens=16000`.** Haiku 4.5 supports extended thinking but its useful budget envelope is smaller than Sonnet 4's. Worth confirming a quick local run that the streaming path doesn't OOM-truncate the thinking block; if it does, drop the budget to 8000 in the same PR.
4. **No mention of running the listed `uv run pytest` commands.** The "Test plan" checkboxes are all unchecked. Either run them and tick the boxes, or note that they require a live Anthropic key the contributor doesn't have so a maintainer needs to validate before merge.
