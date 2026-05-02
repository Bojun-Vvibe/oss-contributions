# BerriAI/litellm PR #27034 — fix(proxy): wire general_settings url-validation to litellm globals

- **URL**: https://github.com/anomalyco/litellm/pull/27034
- **Head SHA**: `ee3e812186e5`
- **Files touched**: `litellm/proxy/proxy_server.py` (~9 lines), `tests/proxy_unit_tests/test_proxy_config_unit_test.py` (~57 lines)

## Summary

Closes a regression where `user_url_validation` and `user_url_allowed_hosts` set under `general_settings:` in proxy config YAML were parsed but never propagated to the `litellm.*` module globals that `url_utils.validate_url` / `safe_get` / `async_safe_get` actually read at runtime. Patch reads both keys inside `load_config(...)` and assigns them to the module attributes.

## Comments

- `proxy_server.py:4063-4068` — `litellm.user_url_validation = bool(_user_url_validation)` runs `bool(...)` on the YAML-parsed value. YAML already returns `True`/`False` for booleans; the `bool()` cast is defensive but means a string `"false"` from a misconfigured YAML would silently become `True`. Validate type and emit a clear error instead, mirroring how nearby settings handle this.
- `proxy_server.py:4067-4069` — `litellm.user_url_allowed_hosts = list(_user_url_allowed_hosts)` will throw `TypeError` for a YAML scalar (`user_url_allowed_hosts: 10.80.1.20` instead of a list). Fine, but the error will surface deep in startup with little context — wrap with a typed validation that lists the offending key.
- The test (`test_proxy_config_unit_test.py:331-381`) saves and restores `litellm.user_url_validation` / `litellm.user_url_allowed_hosts` in a `try/finally` — good. But it doesn't assert the *previous* state is unchanged on partial-config (e.g. only `user_url_validation` set, no `user_url_allowed_hosts`). Add a second test for the partial case so a future refactor that "always overwrites" gets caught.
- The PR ties this to issue #26599 in the docstring (`test_proxy_config_unit_test.py:336`). Worth confirming the issue is public and that referenced behaviour matches — minor doc hygiene.
- The new wiring sits inside the broad `load_config` function (`proxy_server.py:4060+`). That function already has a `noqa: PLR0915` (too many statements). Adding two more settings without extracting a helper is mildly anti-pattern; consider a `_apply_url_validation_settings(general_settings)` helper.
- Behaviourally trivial — two assignments. Low blast radius. The fix is the right shape.

## Verdict

`merge-as-is` — small, well-tested regression fix that restores documented behaviour. The nits above are polish, not blockers.
