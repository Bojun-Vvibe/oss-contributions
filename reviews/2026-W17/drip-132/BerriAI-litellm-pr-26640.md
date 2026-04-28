# BerriAI/litellm PR #26640 — fix(proxy): handle OAuth error in SSO callback

- **PR**: https://github.com/BerriAI/litellm/pull/26640
- **Author**: @skgandikota
- **Head SHA**: `8248f42418a8497fdf0bd921f75ea15804b37a94`
- **Size**: +94 / −0 across 2 files (1 prod, 1 test)
- **Closes**: #26403

## Summary

Closes a real crash in `/sso/callback` where the IdP redirects back with `?error=access_denied&error_description=...` (e.g. user clicked "Cancel" on the consent screen) instead of the expected `?code=...` parameter. Prior to this fix, `auth_callback` proceeded into the normal token-exchange code path which then crashed with a non-graceful error trying to parse the absent `code` parameter, surfacing as a 500 to the user instead of a documented 401. The fix adds a precondition check at the top of the handler that converts an `error` query param into an `HTTPException(401)` with the OAuth error class and description embedded in the detail string.

## Verdict: `merge-as-is`

Textbook fail-fast shape for an OAuth callback handler. The check is at the top of the function (lines 1311-1326) before any state validation or token-exchange code runs, the `isinstance(oauth_error, str) and oauth_error` guard correctly rejects both the missing-key case (`None`) and the empty-string case (`""`), the description-formatting handles the optional-description case without producing the literal substring `"None"` in the user-visible detail, and the test file at `tests/test_litellm/proxy/management_endpoints/test_ui_sso.py:5512-5589` covers the three cells that matter (error-with-description, error-without-description, no-error-passes-through) with the no-error case having a strong "doesn't fire and downstream proceeds normally" assertion.

## Specific references

- `litellm/proxy/management_endpoints/ui_sso.py:1311-1312` — extracts both `error` and `error_description` from `request.query_params`. Both extracted before any validation so the warning log line at `:1318-1321` can include both even if only `error` is set. Correct shape.
- `litellm/proxy/management_endpoints/ui_sso.py:1314` — `isinstance(oauth_error, str) and oauth_error` is the right discriminator: `request.query_params.get("error")` returns `None` for missing key, `""` for `?error=`, and the actual string otherwise. The combined check rejects both `None` (because `isinstance(None, str)` is False) and `""` (because the second clause is falsy). Without the `isinstance` guard, a Starlette `MultiDict` returning a non-string for some reason would produce a `TypeError` on the truthiness check; the explicit isinstance is defensive and cheap.
- `litellm/proxy/management_endpoints/ui_sso.py:1316-1319` — `_err_desc` follows the same `isinstance` pattern for the description, then is used both in the warning log and in the conditional detail-string suffix. The `+ (f", error_description: {_err_desc}" if _err_desc else "")` pattern at `:1325` is the right shape for "include only if present" — the alternative of f-string interpolation with a default would print `"description: None"` to the end user, which is unhelpful and the test at `:5559` asserts `"None" not in str(exc_info.value.detail)` to lock that out.
- `litellm/proxy/management_endpoints/ui_sso.py:1322-1326` — `HTTPException(status_code=401, ...)` is the correct status code per OAuth 2.0 RFC 6749 §4.1.2.1 for authorization-code flow errors returned via redirect. (A case could be made for 400 since the *callback* request itself is well-formed, but 401 matches the user-facing intent of "you are not authorized to log in" and matches what Starlette/FastAPI tutorials recommend for SSO callback failure paths.)
- `tests/test_litellm/proxy/management_endpoints/test_ui_sso.py:5519-5530` — `test_auth_callback_raises_on_oauth_error` is the positive-path test, asserts both the error class string and the description string survive into `exc_info.value.detail`. **Strong**: a future change that drops the `error_description` from the detail string fails this test.
- `tests/test_litellm/proxy/management_endpoints/test_ui_sso.py:5546-5560` — `test_auth_callback_omits_error_description_when_absent` is the regression anchor for the "no None substring" invariant — exactly the right negative assertion for this shape of fix.
- `tests/test_litellm/proxy/management_endpoints/test_ui_sso.py:5562-5589` — `test_auth_callback_does_not_raise_when_no_oauth_error` is the load-bearing negative test that proves the new guard doesn't fire on the normal path. Mocks the cli-callback path so the assertion `mock_cli_callback.assert_called_once()` confirms execution actually reached past the guard. Without this test, a future "let's also catch missing-code as an error" overreach could regress silently.

## Nits (none blocking)

1. The detail string `f"OAuth error: {oauth_error}, error_description: {_err_desc}"` interpolates *attacker-controlled* values into the response body. While these values come from the IdP redirect (not directly from the user), an attacker who can MitM the redirect URL or get a malicious IdP could plant content here. Consider truncating each to ~200 chars and HTML-escaping, since the detail string ends up in browser-rendered error pages.
2. The `verbose_proxy_logger.warning` at `:1318-1321` uses `!r` formatting which is correct for log fields, but logs the description verbatim — same attacker-controlled-content concern, though log injection is generally lower risk than HTML rendering. Cap at a reasonable length.
3. Consider also handling the OAuth `error_uri` field (RFC 6749 §4.1.2.1) — IdPs sometimes return a URL pointing to a help page explaining the error class. Including that URL in the detail (with allowlist validation) would improve self-service debugging.
