# Review: BerriAI/litellm#26766 — Implement Azure AD token authentication for image edit

- **PR:** https://github.com/BerriAI/litellm/pull/26766
- **Head SHA:** `a1b254f263d539bce98e8760439d28df61c63e82`
- **Author:** FredericKayser
- **Diff size:** +129 / -18 across 2 files
- **Verdict:** `merge-after-nits`

## What it does

Fixes a real bug: `litellm/llms/azure/image_edit/transformation.py:validate_environment` was hardcoded to set only `Authorization: Bearer ${api_key}` regardless of whether the caller provided an API key, an `azure_ad_token`, or an `azure_ad_token_provider`. Azure OpenAI accepts API keys via the `api-key` header (NOT `Authorization: Bearer`), and accepts AD tokens via `Authorization: Bearer`. The previous code shoved API keys into the Bearer slot, which Azure rejects with 401 for the API-key auth path.

Fix: delegates to `BaseAzureLLM._base_validate_azure_environment(headers, GenericLiteLLMParams)` which already implements the correct precedence chain (api_key → api-key header; azure_ad_token / azure_ad_token_provider → Authorization: Bearer; managed identity fallthrough). Net deletion of ~12 lines of duplicated/wrong logic.

Tests:
- Updated existing `test_image_edit` assertion to expect `api-key` header (not `Authorization`) when API key is provided.
- New `test_azure_image_edit_azure_ad_token_auth` — passes `azure_ad_token_provider` lambda, asserts `Authorization: Bearer fake-azure-ad-token-12345` is set AND `api-key` is NOT present.
- New `test_azure_image_edit_api_key_takes_precedence` — passes BOTH `api_key` and `azure_ad_token_provider`, asserts `api-key` header wins and `Authorization` is absent.

## Specific citations

- `litellm/llms/azure/image_edit/transformation.py:22-28` — the new body is 6 lines (instantiate `GenericLiteLLMParams`, override `api_key` if explicitly passed, call `_base_validate_azure_environment`). This is the right shape: image-edit is one of N Azure endpoints and they should all funnel through the same auth helper.
- `litellm/llms/azure/image_edit/transformation.py:30-37` (deleted) — the old `headers.update({"Authorization": f"Bearer {api_key}"})` was the bug.
- `tests/image_gen_tests/test_image_edits.py:329-335` — the assertion flip from `assert "Authorization" in headers` → `assert "api-key" in headers and headers["api-key"] == test_api_key` is the load-bearing regression-protection.
- `tests/image_gen_tests/test_image_edits.py:783-840` (new test 1) — discriminating because it asserts BOTH the positive (`Authorization` present with the expected token) AND the negative (`api-key` NOT present).
- `tests/image_gen_tests/test_image_edits.py:842-899` (new test 2) — the precedence test correctly proves api_key wins when both are provided. Worth adding the symmetric "azure_ad_token (string, not provider) wins when no api_key" case.

## Nits to address before merge

1. **Add a fourth test for the `azure_ad_token` (eager string, not callable provider) path.** The current tests cover api_key-only, ad_token_provider-only, and both-with-api-key-winning. Missing: `azure_ad_token="static-token"` alone. `_base_validate_azure_environment` likely handles all three uniformly, but pinning the third path with an explicit test prevents future regressions in the helper from silently breaking image-edit.
2. **Add an "azure_ad_token_provider raises" test.** What happens if the lambda throws? Current code presumably propagates the exception; a test asserting that propagation prevents a future "swallow auth errors" defect from making it back into the image-edit code path.
3. **Verify `_base_validate_azure_environment` handles managed identity (DefaultAzureCredential).** The PR's stated goal includes "delegating to `BaseAzureLLM._base_validate_azure_environment`" which suggests it does — but a one-line comment in the PR body confirming "MSI/managed identity now works for image-edit because $reason" would help reviewers who don't know the helper internals.
4. **Existing `test_image_edit` mutation is a behavior change for any downstream consumer asserting on the old (wrong) `Authorization: Bearer` header.** This is a bug fix, not a regression, but the CHANGELOG should call out: "Azure image edit now correctly uses `api-key` header for API-key auth (was previously sending API key in `Authorization: Bearer` header, which Azure rejected with 401)."
5. **`if api_key: _litellm_params.api_key = api_key`** at `transformation.py:24-25` — this means `api_key=""` (empty string) won't override. Probably fine (empty string isn't a valid key) but worth a comment if `_base_validate_azure_environment` expects `None` vs empty distinctly.
6. **Test imports `from litellm import aimage_edit` inside the test bodies** rather than at the top. Style nit only — moves the import cost into every test invocation but matches the file's existing style based on the diff context.

## Rationale

This is a bug fix masquerading as a small refactor. The duplicated auth logic in `image_edit/transformation.py` was wrong (Bearer-shaped for API keys), and the fix correctly funnels through the canonical Azure helper. The tests are discriminating in the most important way — they assert both the present-header AND the absent-header, which is what catches future "set both headers to be safe" regressions. Two missing test cases (eager `azure_ad_token` string + provider-raises) and a CHANGELOG note are the only blockers.