# BerriAI/litellm PR #27008 — fix(auth): support JWT issuer verification + warn when unscoped

- PR: https://github.com/BerriAI/litellm/pull/27008
- Head SHA: `e55401e39cdc6798738bee031d0e3276310f6425`
- Author: @stuxf
- Size: +155 / -7

## Summary

`auth_jwt` was effectively disabling audience verification when `JWT_AUDIENCE` was unset (`options={"verify_aud": False}`) and never verifying issuer at all. Combined with the proxy's "no team_object / no user_object" default-allow branch, this means tokens minted by *any* application sharing the same IdP signing keys (Azure AD, Okta, etc.) — not just the proxy's intended app — would be accepted. Real cross-tenant escape.

Fix introduces:
- `JWT_ISSUER` env var → when set, PyJWT verifies the `iss` claim.
- A class-level `_unscoped_jwt_warning_emitted` sentinel so the proxy logs exactly one warning per process when both vars are unset.
- A `_build_decode_kwargs` classmethod that centralizes the audience/issuer/options resolution previously duplicated across the RSA/EC/OKP path and the x509 path.

Backward compatible: no env vars set → same behavior as today (signature + expiry only), with a warning.

## Specific references from the diff

- `litellm/proxy/auth/handle_jwt.py:708-748` — the new `_build_decode_kwargs` classmethod. Pulls `JWT_AUDIENCE` + `JWT_ISSUER`, builds the `verify_aud` / `verify_iss` opt-out map only for the unset ones, and emits the once-per-process warning. Returns `{"audience", "issuer", "options"}` with `options=None` when nothing needs to be opted out.
- `litellm/proxy/auth/handle_jwt.py:749-788` — `auth_jwt` now calls `decode_kwargs = self._build_decode_kwargs()` once and passes `**decode_kwargs` to both `jwt.decode` call sites (RSA/EC/OKP at line 786, x509 at line 813). Removes the duplicated `audience=audience, options=decode_options` kwargs at both sites.
- `tests/test_litellm/proxy/auth/test_handle_jwt.py:2570-2680` — six new tests:
  - `_reset_unscoped_warning_flag` autouse-style fixture that resets the class sentinel.
  - `test_build_decode_kwargs_no_env_disables_both_verifications` — both opt-outs present, both env vars None.
  - `test_build_decode_kwargs_audience_only_enables_aud_verification` — only `verify_iss: False` in options.
  - `test_build_decode_kwargs_issuer_only_enables_iss_verification` — symmetric.
  - `test_build_decode_kwargs_both_set_enables_full_verification` — `options is None`.
  - `test_build_decode_kwargs_warns_once_when_unscoped` — three calls, exactly one warning record.
  - `test_build_decode_kwargs_no_warning_when_scoped` — `JWT_AUDIENCE` set → zero warnings.

## Verdict: `merge-after-nits`

This is a real cross-tenant security fix and the implementation is clean: one helper, two call sites, six focused tests covering the truth table. Default behavior preserved with a flag. The only things keeping this off `merge-as-is` are operational concerns around the warning, not the code.

## Nits / concerns

1. **Class-level sentinel `_unscoped_jwt_warning_emitted` is per-process, not per-instance.** This is correct for the "log once per startup" intent, but in pytest with multiple `JWTHandler` instances spread across test files, one test that triggers the warning will silence it for everything that follows in the same process. The `_reset_unscoped_warning_flag` fixture handles this *within* `test_handle_jwt.py`, but any other test file that constructs a `JWTHandler` and inspects logs is now at risk of flakiness. Worth a short docstring on the sentinel noting "tests must reset this between cases" or moving it to instance-level with a process-wide `Lock`-guarded counter.
2. **Warning fires even when the operator has *deliberately* chosen unscoped JWT** (e.g. dev environments, local docker-compose). Consider: (a) make the warning gateable via `LITELLM_SUPPRESS_JWT_UNSCOPED_WARNING=true`, or (b) downgrade to `info` level after the first 24 hours (harder). As written, ops teams will see this on every restart and either tune it out or grep their config for `JWT_AUDIENCE` and set it to a junk value to silence — defeating the purpose.
3. **`options or None`** at `handle_jwt.py:746` is subtle: an empty dict becomes `None` so PyJWT uses defaults. That's correct (PyJWT default-verifies both claims when audience+issuer are passed), but a one-line comment "PyJWT defaults verify_aud/verify_iss to True when their respective claims are passed" would help the next reader.
4. **No integration test that actually decodes a token with mismatched `iss`.** The six new tests verify `_build_decode_kwargs` returns the right shape, but nothing exercises an end-to-end `auth_jwt(token)` call where `iss` is wrong and PyJWT raises `InvalidIssuerError`. The PR description claims this was smoke-tested manually; please bake one such test into the same file with a mocked JWKS and a real PyJWT-encoded token.
5. **`leeway` kwarg disappears from the x509 path.** Looking at lines 745-746 vs. 805-810: the RSA/EC/OKP path keeps `leeway=self.leeway` *plus* `**decode_kwargs`, while the x509 path is `**decode_kwargs` only. Was leeway intentionally removed from x509? If yes, document. If no, restore it.
