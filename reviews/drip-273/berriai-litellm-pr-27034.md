# BerriAI/litellm PR #27034 — fix(proxy): wire general_settings url-validation settings to litellm globals

- **PR**: https://github.com/BerriAI/litellm/pull/27034
- **Head SHA**: `ee3e812186e56a574a6e289cc8f2b9134c5396fc`
- **Size**: +63 / −0 across 2 files (`litellm/proxy/proxy_server.py`, `tests/proxy_unit_tests/test_proxy_config_unit_test.py`)
- **Verdict**: **merge-after-nits**

## What changed

`proxy_server.py` `load_config` (around line 4060) gains a new "URL VALIDATION SETTINGS" block that reads `user_url_validation` and `user_url_allowed_hosts` out of `general_settings` and assigns them to `litellm.user_url_validation` (coerced via `bool()`) and `litellm.user_url_allowed_hosts` (coerced via `list()`). The block uses the same `general_settings.get(..., None)` + None-check pattern as the surrounding `proxy_budget_rescheduler_min_time` / `allowed_ips` settings. A regression test `test_general_settings_url_validation_wired_to_litellm` writes a temp YAML config with `user_url_validation: False` and `user_url_allowed_hosts: ["10.80.1.20", "internal.corp"]`, calls `ProxyConfig().load_config(...)`, and asserts that `litellm.user_url_validation is False` and `litellm.user_url_allowed_hosts == [...]`. The test snapshots and restores the original module globals in a `finally:` block.

## Why this matters

`litellm.user_url_validation` and `litellm.user_url_allowed_hosts` are consumed by `url_utils.validate_url` / `safe_get` / `async_safe_get` (per the test docstring's reference to issue #26599). Before this change, setting these in `general_settings` had no effect — they had to be set as environment variables or via direct module assignment, which is a config-surface-area bug because the YAML file is the documented configuration entry point. This is the standard "config option silently does nothing" class of bug, and the fix is the minimal correct one: lift the values out of `general_settings` and stuff them into the module globals where the consumers read them.

The default-disabled behavior (`if … is not None`) is correct — leaving the globals untouched when the user didn't set the key preserves backward compatibility for anyone relying on env-var or programmatic init.

## Specific call-outs

- `proxy_server.py` line ~4063: `litellm.user_url_validation = bool(_user_url_validation)`. The `bool()` coercion turns `"false"` (string from sloppy YAML) into `True` because non-empty strings are truthy. YAML normally produces a real `False`, but if anyone uses an env-substitution string like `${URL_VALIDATION}` and it lands as `"false"`, this silently flips. **Nit**: use the same pattern as elsewhere in this file, or explicitly handle the string→bool case. At minimum, document that this expects a real YAML boolean.
- `litellm.user_url_allowed_hosts = list(_user_url_allowed_hosts)`: `list()` on a string iterates characters. If a user accidentally writes `user_url_allowed_hosts: "internal.corp"` (string instead of list), the global ends up as `['i','n','t',...]`. **Nit**: add an `isinstance(_user_url_allowed_hosts, (list, tuple))` check with a clear error, mirroring how `allowed_ips` is validated above.
- The test correctly snapshots `original_validation` and `original_hosts` and restores in `finally:` — important because module globals leak across tests in the same process. Good practice.
- **Nit**: the test asserts the happy path only. Add a second test (or parametrize) for the "key absent → globals untouched" case to lock in the no-op behavior. Without it, a future refactor that always overwrites the globals (even on `None`) would not be caught.
- The reference to "VERIA-…" or any internal ticket isn't in this PR (good — kept clean), but the issue link `https://github.com/BerriAI/litellm/issues/26599` in the test docstring is useful provenance.
- Placement of the new block between `allowed_ips` and `proxy_budget_rescheduler_min_time` is reasonable; it groups with other security/access settings.

## Verdict rationale

Minimal, surgical, test-covered fix for a real "configuration silently ignored" bug. Both nits are about input validation hardening, not the core wiring — the wiring itself is correct. Land after either tightening the type coercions or adding a follow-up test for the absent-key path. No security risk introduced; if anything this *enables* the user to use the documented URL validation feature.
