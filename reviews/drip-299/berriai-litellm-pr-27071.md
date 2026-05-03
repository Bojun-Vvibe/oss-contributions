# BerriAI/litellm PR #27071 — chore(proxy): drop client-supplied pricing fields from request bodies

- URL: https://github.com/BerriAI/litellm/pull/27071
- Head SHA: `4b4b4a79b9216d7cd808f4bb277ac8223359dd52`
- Author: stuxf

## Summary

Adds `_strip_client_pricing_overrides()` in
`litellm/proxy/litellm_pre_call_utils.py` that, by default, removes every
field declared on `CustomPricingLiteLLMParams` (`input_cost_per_token`,
`output_cost_per_token`, etc.) plus `metadata.model_info` and
`litellm_metadata.model_info` from the inbound request body before the
proxy hands it to `litellm.completion`. An opt-in
`allow_client_pricing_override: true` flag on the calling key/team
metadata preserves the legacy behavior. Ships with
`tests/test_litellm/proxy/test_pricing_field_strip.py` (~312 lines).

## Review

This is a real security/billing fix — without it, any client could send
`{"input_cost_per_token": 0}` and not only zero out their own usage log,
but via `litellm.register_model` mutate the worker-wide
`litellm.model_cost` map for *every* subsequent call. The PR description
in the diff comment block names this attack chain explicitly, which is
nice.

Implementation notes:

- `_CLIENT_PRICING_CONTROL_FIELDS = frozenset(CustomPricingLiteLLMParams
  .model_fields.keys())` is the right move — auto-covers any field added
  to that Pydantic model later. Add a regression test that *fails* if a
  new field is added without bumping the test fixture, otherwise this
  silently drifts.
- Strip is placed **after** the `litellm_metadata` JSON-string parse
  (`add_litellm_data_to_request` ~line 1370 in the diff), with a
  comment explaining why. Correct — earlier strip would miss
  multipart/form-data callers.
- `_strip_client_pricing_overrides` mutates `data` in place and logs at
  `debug`. Consider bumping to `info` (or at least `warning` once per
  key/minute) so operators can notice misconfigured clients without
  having to enable debug logging proxy-wide.
- The `allow_client_pricing_override` opt-in checks both key and team
  metadata via `_key_or_team_metadata_flag_is_true`, which matches the
  existing `allow_client_message_redaction_opt_out` and `allow_client_tags`
  precedents. Good consistency.

Verify before merge: any internal caller (LiteLLM admin UI, virtual key
management, etc.) that legitimately sets `input_cost_per_token` per request
needs the opt-in flag set on its key, or its requests will silently start
charging at the model-default rate.

## Verdict

`merge-after-nits` — promote the strip log to `info`/structured, add a
guard test that asserts every `CustomPricingLiteLLMParams` field is
exercised, and document the opt-in flag in the proxy admin docs.
