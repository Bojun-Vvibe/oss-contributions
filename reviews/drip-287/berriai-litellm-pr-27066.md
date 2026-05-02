# BerriAI/litellm PR #27066 — fix(proxy): strictly honor disable_end_user_cost_tracking in SpendLogs

- Head SHA: `7cc9f6787a7c2dc74b8dfe5842703c129c09a9c3`
- URL: https://github.com/BerriAI/litellm/pull/27066
- Size: +9 / -3, 2 files
- Verdict: **merge-after-nits**

## What changes

Two surgical fixes that close the same gap from different angles:

1. `litellm/proxy/auth/auth_utils.py:855-859` — `get_end_user_id_from_request_body`
   now early-returns `None` if `litellm.disable_end_user_cost_tracking is True`
   *before* doing any header/body parsing. This prevents the end-user ID
   from ever leaving the request layer when the operator has explicitly
   opted out of per-end-user attribution.
2. `litellm/proxy/spend_tracking/spend_tracking_utils.py:298-301` — in
   `get_logging_payload`, the existing fallback that pulled `end_user_id`
   from `metadata["user_api_key_end_user_id"]` is now gated behind the
   same flag. Previously this was the leak path: even with
   `disable_end_user_cost_tracking=True` set, SpendLogs rows could still
   end up with an `end_user` populated from auth metadata.

## What looks good

- Defense-in-depth at both the entry point (auth) and the persistence
  point (spend tracking). Either one alone would have left a hole.
- The condition uses `is True` (identity check), which matches existing
  litellm convention for tri-state flags (`True` / `False` / `None`)
  and avoids accidentally treating a truthy non-bool as opt-in.
- Diff size is tiny — easy to backport.

## Nits

1. `import litellm` happens inside the function body in
   `auth_utils.py`. The file already imports `litellm` at module scope
   in most other places — confirm there's a real circular-import reason
   here, otherwise hoist the import for consistency. (If circular, a
   one-line comment would help future maintainers not "fix" it.)
2. No new test asserting that `end_user_id` is `None` in the resulting
   SpendLogs row when `disable_end_user_cost_tracking=True`. Given this
   is a privacy/compliance-flavored fix, a regression test guarding
   both code paths would be high-value. Look at
   `tests/proxy_unit_tests/test_proxy_setting_guardrails.py` for a
   similar pattern.
3. The change in `spend_tracking_utils.py` only guards the
   `metadata["user_api_key_end_user_id"]` fallback. Confirm there's no
   *third* path (e.g. via `request_body["user"]` or `litellm_metadata`)
   that can still populate `end_user_id` downstream of this function.

## Risk

Low. Behavior change only fires when the operator has explicitly
toggled the flag, and the change is "do less" not "do more".
