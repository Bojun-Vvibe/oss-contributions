# BerriAI/litellm #27235 — fix(sso): /sso/debug/callback shows full JWT claims + parsed fields

- URL: https://github.com/BerriAI/litellm/pull/27235
- Head SHA: `c06657e2bd9660a560fedb60ab8988cdd6944989`
- Diff: +159 / -53 across 2 files (`litellm/proxy/management_endpoints/ui_sso.py` +85/-24; `litellm/proxy/common_utils/html_forms/jwt_display_template.py` +74/-29)

## Findings

1. The original bug is real and well-diagnosed. At `litellm/proxy/management_endpoints/ui_sso.py` the generic-SSO branch was unpacking `result, _, _ = get_generic_sso_response(...)` and silently discarding the raw `received_response` — so operators couldn't see what the IdP actually returned, only what LiteLLM mapped. The HTML template at `jwt_display_template.py` (around the deleted lines `if typeof value !== 'object' || value === null`) was filtering out lists and dicts entirely, which silently dropped `team_ids` (list) and `extra_fields` (dict) from the rendered page. Both bugs fixed.
2. New helper `_to_plain_dict(obj)` in `ui_sso.py` (around line 446 in the diff) correctly handles three input shapes: plain `dict`, pydantic objects (`hasattr(obj, "model_dump")`), and arbitrary objects (`__dict__` / `str()` fallback). Filtering of `_`-prefixed private fields from `model_dump` output is sensible.
3. **Security review — CRITICAL**: `raw_claims` now ships the **complete** unmodified IdP claim set to the browser, including `iss`, `aud`, `exp`, `iat`, and any custom claims. For most IdPs this is fine (the user already authenticated). However:
   - If the IdP returns the raw `id_token` string in `received_response`, `_to_plain_dict` will include it. An `id_token` is a bearer credential; rendering it on a debug page is a token-leak vector if the operator screen-shares the page. Recommend explicitly stripping known-sensitive keys (`access_token`, `id_token`, `refresh_token`, `at_hash`, `c_hash`) from `raw_claims` before sending to the template.
   - The endpoint is named `/sso/debug/callback` and presumably gated by `SSO_DEBUG_MODE` or similar, but the PR description doesn't confirm this. If this route is reachable in production without an admin guard, it's a serious data-exposure regression. Please confirm the route's auth guard and either add one or make the gating explicit.
4. JS `formatValue` / `renderInto` rewrite at `jwt_display_template.py:233+` is clean and the legacy-flat-payload backwards-compat branch (`Object.prototype.hasOwnProperty.call(userData, 'parsed_by_proxy')`) is correctly defensive.
5. No new tests. Given this is a security-adjacent endpoint that now exposes more data, at least one test asserting that the response shape contains `parsed_by_proxy` and `raw_claims` keys, and that `_to_plain_dict` correctly preserves nested structures, would be appropriate.

## Verdict

**Verdict:** request-changes
