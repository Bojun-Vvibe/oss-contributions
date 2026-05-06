# BerriAI/litellm#27263 — fix(snowflake): update Cortex endpoint from inference:complete to v1/chat/completions

- **Head SHA**: `ea666010184c2b75956df384ddf098324b89281c`
- **Author**: VANDRANKI (PRABHU KIRAN VANDRANKI)
- **Stats**: +3 / -3, 2 files (fixes #27187)

## Summary

Snowflake retired their proprietary `/api/v2/cortex/inference:complete` endpoint in favor of the OpenAI-compatible `/api/v2/cortex/v1/chat/completions`. PR updates the URL builder + a stale docstring + a test assertion. Also resolves a double-path bug where users supplying an `api_base` already containing `/cortex/v1` would get an invalid concatenated URL.

## Specific citations

- `litellm/llms/snowflake/chat/transformation.py:155` (in `ea66601` at `+`): `return f"{api_base}/cortex/inference:complete"` → `return f"{api_base}/cortex/v1/chat/completions"`. One-line surgical change.
- `transformation.py:150` corrects stale docstring `"default DeepSeek /chat/completions endpoint"` → `"default Snowflake Cortex /chat/completions endpoint"` — lingering copy-paste from the original DeepSeek-cloned file.
- `tests/test_litellm/llms/snowflake/chat/test_snowflake_chat_transformation.py:396`: `assert post_kwargs["url"].endswith("cortex/inference:complete")` → `assert post_kwargs["url"].endswith("cortex/v1/chat/completions")` — the JWT account-id test.

## Verdict

**merge-after-nits**

## Rationale

Endpoint change is straightforward and fixes a real outage class (the old URL is presumably 404'ing on the upstream side now). The PR claim about the double-path bug (`api_base` ending in `/cortex/v1` getting `/api/v2` re-appended) is plausible but **not visibly tested in this diff** — the only test touched is `test_snowflake_jwt_account_id`. A regression test for the "user-supplied api_base already includes /cortex/v1" case would directly pin the second claim. Also: only the `test_snowflake_jwt_account_id` assertion is updated; the sibling `test_snowflake_pat_key_account_id` test method's URL assertion (typically right next to it) should be checked for the same pattern — if it asserts the old endpoint string, this PR will leave it failing. Reviewer should `rg "cortex/inference:complete"` across the test directory to confirm full coverage. Otherwise this is a minimal, correct fix worth landing fast.

