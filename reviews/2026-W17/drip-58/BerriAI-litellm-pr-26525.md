# BerriAI/litellm#26525 — broaden RAG ingestion credential cleanup to AWS endpoint/identity fields

- **Repo:** BerriAI/litellm
- **Author:** yuneng-berri (yuneng-jiang)
- **Head SHA:** `30cb459d` (30cb459d1c924f84a01bb5338b9c37819946dd21)
- **Size:** +394 / -5

## Summary
`_load_credentials_from_config` previously cleaned up only stale `api_base`
when a stored credential didn't carry one. PR generalizes the cleanup to
also drop `aws_sts_endpoint` and `aws_web_identity_token` so caller-supplied
AWS endpoint/identity values can't leak past credential resolution. Bulk of
the diff is unrelated model-pricing additions in
`model_prices_and_context_window_backup.json`.

## Specific references
- `litellm/rag/ingestion/base_ingestion.py:73-78` @ `30cb459d` — docstring updated from "the api_key / api_base pair" to "endpoint and identity fields". Accurate.
- `litellm/rag/ingestion/base_ingestion.py:90-99` @ `30cb459d` — replaces the single-key delete with a `for key in ("api_base", "aws_sts_endpoint", "aws_web_identity_token")` loop. Logic preserved per-key: only delete when the key is in the (caller-supplied) `vector_store_config` and was NOT provided by the resolved credential. Correct.
- `model_prices_and_context_window.json` and the `_backup.json` — large pricing-catalog additions for `azure/gpt-5.5*` family. Unrelated to the security fix; routinely included in this repo's PRs.

## Observations
1. **AWS credential surface is wider than 3 fields**: `aws_session_token`, `aws_role_name`, `aws_profile_name`, `aws_region_name` can also be supplied via the same `vector_store_config` and could similarly mask a stored credential. The PR fixes 3 specific fields but leaves the same class of bug for siblings. Either expand the tuple to cover the full Bedrock/AWS auth keys, or refactor: "delete every caller-supplied key that the credential doesn't redefine and that's auth-related." A constant `AWS_AUTH_FIELDS` would let the test assert coverage by introspection.
2. **No targeted unit test in the diff** for the new `aws_sts_endpoint` / `aws_web_identity_token` cleanup. The PR touches `tests/test_litellm/test_utils.py` and `test_llm_cost_calc_utils.py` — but those look pricing-related (matching the bulk of the diff). A 10-line test that constructs a `vector_store_config` with `aws_sts_endpoint` set, resolves a stored credential lacking it, and asserts the field is gone would lock the fix.
3. **Mixing pricing-catalog and security fix in one PR** makes audit harder. Reviewers who skim the title may merge without noticing the security-relevant change is hidden under 380 lines of JSON. Splitting would be cleaner, though it's a recurring pattern in this repo.

## Verdict
**merge-after-nits** — add the targeted test for the new fields; consider expanding the AWS field list. The fix itself is correct.
