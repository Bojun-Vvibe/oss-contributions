# BerriAI/litellm PR #27069 — fix(proxy): honor disable_end_user_cost_tracking in spend logs

- URL: https://github.com/BerriAI/litellm/pull/27069
- Head SHA: `3623c446ac46a09ca55a363509b3756dcf16cd06`
- Author: fengfeng-zi
- Repo: BerriAI/litellm

## Summary

When `disable_end_user_cost_tracking` is set, `get_logging_payload` was still
falling back to `metadata.user_api_key_end_user_id` from the standard logging
payload, leaking the end-user id into spend logs. This PR gates that fallback
behind the flag.

## Specific notes

- `litellm/proxy/spend_tracking/spend_tracking_utils.py:298-301` — the new
  conditional wraps only the `standard_logging_payload` fallback. There is
  still an earlier read (around line 282 in the existing function) where
  `end_user_id` is initialized from `kwargs`/proxy metadata; the PR does
  **not** also gate that path. If the upstream caller already populates
  `end_user_id` from request metadata, the flag will still be ignored.
  Recommend either gating the entire `end_user_id` resolution block, or
  applying a final `if litellm.disable_end_user_cost_tracking:
  end_user_id = ""` just before the payload is assembled, so the contract
  is enforced once at the boundary regardless of source.
- `tests/.../test_spend_tracking_utils.py:1043-1149` — parametrized test
  covers `(True, "")` and `(False, "test-end-user")`. Good. The test only
  populates `metadata.user_api_key_end_user_id` (the `standard_logging_payload`
  path the diff actually touches); a second test feeding `end_user_id`
  through `kwargs["litellm_params"]["metadata"]` would lock down the broader
  contract and surface the gap above.
- Minor: prefer `not getattr(litellm, "disable_end_user_cost_tracking", False)`
  if there is any chance the attribute is unset in older configs, though
  this codebase reads the attribute directly elsewhere so consistency with
  existing style is fine.

verdict: merge-after-nits
