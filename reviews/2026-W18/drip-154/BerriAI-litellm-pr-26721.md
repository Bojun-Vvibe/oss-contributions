# BerriAI/litellm #26721 — chore(tests): replace deprecated bedrock claude-3-7-sonnet with claude-sonnet-4-5 across test fixtures

- PR: https://github.com/BerriAI/litellm/pull/26721
- Head SHA: `b1a0a3fc17d616ad2993a4c73f4eecbf31ef005c`
- Files: 16 test files under `tests/llm_translation/`, `tests/litellm_utils_tests/`, etc.; production code (`constants.py`, `model_prices_and_context_window.json`) intentionally untouched.

## Citations

- `tests/litellm_utils_tests/test_health_check.py:317-321` — model string in `_update_litellm_params_for_health_check` regression test (the test for #15807 that pins "Bedrock health checks must not send `region/model` as model ID to AWS") flips from `"bedrock/us-gov-west-1/anthropic.claude-3-7-sonnet-20250219-v1:0"` to `"bedrock/us-gov-west-1/anthropic.claude-sonnet-4-5-20250929-v1:0"`. Assertion `assert updated_params["model"] == "anthropic.claude-sonnet-4-5-20250929-v1:0"` — the *region-stripping* contract being tested is unchanged; only the model string the test exercises it with moves forward.
- `tests/litellm_utils_tests/test_utils.py:2309-2317` — parametrized PDF-input support test renamed `test_claude_3_7_sonnet_supports_pdf_input` → `test_claude_sonnet_4_5_supports_pdf_input`, both rows updated, function body unchanged. Pin remains: `assert supports_pdf_input(model) == True` for both bare `anthropic.claude-sonnet-4-5-20250929-v1:0` and the cross-region inference profile `us.anthropic.claude-sonnet-4-5-20250929-v1:0`.
- `tests/llm_translation/test_bedrock_anthropic_regression.py:134-238` — three callsites in `test_prompt_caching_cache_control_transforms_correctly`, `test_prompt_caching_no_beta_header_added`, `test_1m_context_with_prompt_caching` flip the `model="us.anthropic.claude-..."` argument passed to `AmazonConverseConfig.transform_request` and `AmazonAnthropicClaudeConfig.transform_request`. Assertions on output `cache_control` shape, beta-header presence, and 1M-context flag passthrough are unchanged.
- `tests/llm_translation/test_bedrock_completion.py:1326`, `1633` — two `data["model"]` callsites in `test_bedrock_completion_test_2` and `test_bedrock_completion_test_4` updated. Test body otherwise unchanged.
- Pattern across remaining 11 files: same string-substitution shape — every occurrence of `claude-3-7-sonnet-20250219-v1:0` becomes `claude-sonnet-4-5-20250929-v1:0`, no logic changes, no new test cases.

## Verdict

`merge-after-nits`

## Reasoning

This is the right kind of test-modernization PR — `claude-3-7-sonnet-20250219-v1:0` was deprecated by AWS Bedrock and any test exercising it against a live endpoint will start failing as the deprecation grace period closes. The substitution to `claude-sonnet-4-5-20250929-v1:0` keeps the same Anthropic model family (so prompt-caching, cache-control, beta-header semantics are equivalent at the SDK layer) while pointing at a non-deprecated artifact. Importantly the PR confines itself to test fixtures and does *not* touch `model_prices_and_context_window.json` or `constants.py`, which is correct: production model registration must stay backwards-compatible because customers in the wild are still calling the old model ID.

What the PR gets right:
1. **Scope discipline.** 16 files, all under `tests/`, all the same mechanical change. No logic edits, no new assertions, no opportunistic refactoring. This is a one-purpose PR.
2. **The contract being tested is preserved.** The health-check regression test at `test_health_check.py:317` is the most important one — it pins the fix for issue #15807 (Bedrock health checks accidentally sending `region/model` as the model ID to AWS, causing 4xx). The fix is in production code; the test exercises it through a model string. Updating the model string keeps the regression test alive against the deprecation; the *fix* is unaffected.
3. **Function rename matches new content.** `test_claude_3_7_sonnet_supports_pdf_input` → `test_claude_sonnet_4_5_supports_pdf_input` at `test_utils.py:2316` is the right hygiene — the old name would have lied about what it tests, and grep-by-test-name would mislead the next person trying to debug claude-3-7 PDF support.

Nits worth raising before merge (none blocking):

1. **No coverage retained for the deprecated model.** The PR removes the only places that exercise `claude-3-7-sonnet-20250219-v1:0` in the test suite. While the model is deprecated, customers will still send requests with that model ID for some weeks/months, and the *translation layer* (`AmazonConverseConfig.transform_request`) needs to keep working for them. Recommend adding back at least one test (e.g. in `test_bedrock_completion.py`) that pins "the translation layer accepts the deprecated model string and produces a structurally valid request envelope, even if AWS would reject it at runtime." This avoids the class of regression where a future refactor accidentally removes special-case code for `claude-3-7-sonnet` that's still load-bearing in customer prod traffic.

2. **Cross-region inference profile coverage is implicit.** The `us.anthropic.claude-sonnet-4-5-...` form is exercised in `test_utils.py:2316` and `test_bedrock_anthropic_regression.py:137`, but the `eu.` and `apac.` profile prefixes (which Bedrock also exposes for sonnet-4-5) are not. Probably out of scope — but if there's an `is_cross_region_profile()` predicate in production code, this would be the moment to spot-check it doesn't false-negative on the new model string.

3. **`test_bedrock_anthropic_regression.py` should rename the test methods that no longer match their new content.** `test_prompt_caching_cache_control_transforms_correctly` and `test_1m_context_with_prompt_caching` were generic enough to survive, but `test_prompt_caching_no_beta_header_added` was originally written to pin the "claude-3-7 doesn't need the prompt-caching beta header" behavior. claude-sonnet-4-5 has the same beta-header semantics (the header was deprecated for both), so the test still passes — but the test *name* implies model-specific behavior that no longer matches the test body. Either rename to `test_prompt_caching_no_beta_header_added_for_anthropic_models` (truer scope) or split into two parametrized cases.

These are pre-merge nits; merge-as-is is also defensible if the maintainer prefers a separate "preserve deprecated-model translation coverage" follow-up.
