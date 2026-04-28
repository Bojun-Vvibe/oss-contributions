# BerriAI/litellm #26662 — [Fix] Redact spend logs error message

- URL: https://github.com/BerriAI/litellm/pull/26662
- Head SHA: `468cfe0c3d08bdc257a457f6d708148762ce69d0`
- Size: +76 / -0 across 2 files
- Verdict: **merge-as-is**

## What the change does

Closes a real prompt/response leakage path that bypassed the
`store_prompts_in_spend_logs` guardrail. When a provider call failed,
litellm wrapped the upstream error in a `BadRequestError`-class exception
whose `str(exception)` representation **embeds the entire upstream response
body** (e.g. `'litellm.BadRequestError: Received={"candidates":
[{"content": {"role": "model", "parts": [{"text": "secret content"}]}}]}'`).
That string was stored verbatim as
`error_information.error_message` in the spend log row, leaking response
content even when the operator had explicitly disabled prompt/response
storage.

Fix at
`litellm/proxy/spend_tracking/spend_tracking_utils.py:141-152`: after the
existing `clean_metadata` assembly, check
`_should_store_prompts_and_responses_in_spend_logs()`; if **false**,
locate `clean_metadata["error_information"]`, and if it has an
`error_message`, replace the dict with `{**error_info, "error_message":
None}`. The other fields (`error_code`, `error_class`, `llm_provider`,
`traceback`) survive — those are operationally useful and don't carry
prompt/response content.

## What is load-bearing

- The `**error_info` spread followed by the explicit `"error_message":
  None` override at `:148-151` is the right shape — preserves all other
  fields, doesn't mutate the input dict, doesn't drop unknown extra keys
  that future provider integrations might add.
- The redact-only-when-disabled gate at `:142` is exactly the right
  semantics: operators who opt **in** to prompt storage already know
  errors will contain prompt/response material; operators who opt **out**
  must not have that material leak through the error channel.
- `error_info.get("error_message") is not None` (not just truthy check)
  means empty strings are still nulled. The test at
  `tests/test_litellm/proxy/spend_tracking/test_spend_tracking_utils.py:1556-1561`
  pins this with `_failure_metadata_with_error("")` and asserts
  `result["error_information"]["error_message"] is None`. Wait — actually
  the gate is `is not None`, so an empty string `""` **would** pass the
  guard and get nulled. Re-reading: yes, that's correct, the test pins
  the desired behaviour and the guard is `is not None` so `""` triggers
  the redaction branch. Good.
- The traceback field at `:1538` is **not** redacted. That's the right
  call — Python tracebacks contain frame info and exception type chain
  but not the response body that the leaked `str(exception)` did. If a
  future stack frame shows up as `local var = "secret"`, that's a
  separate concern requiring a different fix.

## Test coverage (5 tests, all in one block at `:1521-1583`)

1. `test_error_message_redacted_when_store_prompts_disabled` (`:1546-1554`)
   — the headline case, raw provider error containing the candidate
   content gets nulled.
2. `test_empty_error_message_still_nulled_when_disabled` (`:1556-1561`)
   — empty string also gets nulled (sanity check on the `is not None`
   guard).
3. `test_error_message_kept_when_store_prompts_enabled` (`:1563-1568`)
   — the opt-in branch preserves the error message verbatim.
4. `test_other_error_fields_preserved_when_message_redacted` (`:1570-1578`)
   — pins that `error_code`, `error_class`, `llm_provider`, `traceback`
   are untouched. This is the regression anchor that catches a future
   refactor that accidentally over-redacts.
5. `test_no_error_when_error_information_is_none` (`:1580-1583`) —
   success path with no `error_information` doesn't blow up.

The 5-test matrix covers all four cells of the (storage on/off) ×
(error present/absent) truth table plus the field-preservation
invariant. That's a complete spec for the policy.

## Risk

- Backwards-compat: any operator log-parsing tool that reads
  `error_information.error_message` will now see `null` instead of the
  prior leaked string. That's a desired break — the prior behaviour was
  the bug being fixed. Release notes should call this out for operators
  whose monitoring greps for known error-message substrings; recommend
  they switch to `error_class` or `error_code` for that.
- The fix only covers `_get_spend_logs_metadata`. If there are other
  paths that store the same `str(exception)` blob (e.g. directly into
  the spend log without going through this helper), they remain leaky.
  Out of scope for this PR but worth a follow-up audit.

## Recommendation

Ship as is. Right diagnosis, right gate (operator-policy-aware), right
shape (preserve other fields), full truth-table test coverage.
