# BerriAI/litellm#26694 — fix(proxy): surface generic callback env vars in config API

- **PR**: https://github.com/BerriAI/litellm/pull/26694
- **Author**: @milan-berri
- **Head SHA**: `2661b743331e36df43bd3dee9bac15b1f82b99cd`
- **Base**: `main`
- **State**: OPEN
- **Scope**: +92 / -1 across 2 files (1 src + 1 test)

## Summary

`process_callback` in `litellm/proxy/common_utils/callback_utils.py` is the helper that surfaces a callback's required env vars (the names returned by `CustomLogger.get_callback_env_vars()`) and their values to the proxy config API. Currently, when a value is not present in the user-supplied `environment_variables` dict, it falls back to `None` — even when the env var is set in the actual process environment. For "generic" / "custom_callback_api" callbacks this is the wrong behavior: those callbacks intentionally read `GENERIC_LOGGER_ENDPOINT` / `GENERIC_LOGGER_HEADERS` straight from `os.environ` at runtime, so the config API was reporting `null` for env vars that actually had values driving live behavior. The fix adds a 2-element allowlist `GENERIC_API_ENV_VARS = {"GENERIC_LOGGER_ENDPOINT", "GENERIC_LOGGER_HEADERS"}` and routes those names through `os.getenv(_var)` on the missing-from-config path. Config-supplied values still win over env (precedence is preserved).

## Diff anchors

- `litellm/proxy/common_utils/callback_utils.py:1` — adds `import os` at module top.
- `litellm/proxy/common_utils/callback_utils.py:18` — adds module-level `GENERIC_API_ENV_VARS = {"GENERIC_LOGGER_ENDPOINT", "GENERIC_LOGGER_HEADERS"}` constant. Set semantics for O(1) `in` check.
- `litellm/proxy/common_utils/callback_utils.py:531-538` — the load-bearing change inside `process_callback`:
  ```
  if env_variable is None:
      if _var in GENERIC_API_ENV_VARS:
          env_vars_dict[_var] = os.getenv(_var)
      else:
          env_vars_dict[_var] = None
  else:
      env_vars_dict[_var] = env_variable
  ```
  The `else: env_vars_dict[_var] = env_variable` branch at `:537-538` is preserved unchanged, which is what makes "config wins over env" correct: the env-fallback only fires when `environment_variables.get(_var, None) is None`. If config provided an empty string for the env var, `env_variable is None` is false and the empty string is used (this is arguably correct — caller was explicit).
- `tests/test_litellm/proxy/common_utils/test_callback_utils.py:79-104` — `test_process_callback_generic_api_falls_back_to_os_env`: sets the two env vars via `monkeypatch.setenv`, calls with `_callback="generic_api"` and `environment_variables={}`, asserts both values come from os.environ.
- `tests/test_litellm/proxy/common_utils/test_callback_utils.py:107-130` — `test_process_callback_custom_callback_api_falls_back_to_os_env`: same shape with `_callback="custom_callback_api"`. Two-cell coverage of the two callback names that ship the generic logger pattern.
- `tests/test_litellm/proxy/common_utils/test_callback_utils.py:133-158` — `test_process_callback_config_value_wins_over_os_env`: sets env to `https://env.example.com`, passes config to `https://config.example.com`, asserts config wins. This is the load-bearing precedence test — without it, a future refactor could silently flip the precedence and break enterprise deployments that override env via their proxy config.
- `tests/test_litellm/proxy/common_utils/test_callback_utils.py:161-176` — `test_process_callback_does_not_fallback_for_non_allowlisted_env_vars`: sets `NON_ALLOWLISTED_ENV_VAR=secret` in env, asserts the result is `{"NON_ALLOWLISTED_ENV_VAR": None}`. This is the load-bearing **negative** test — it pins the security boundary that the allowlist is doing actual filtering, so a typo widening of the allowlist would surface here. Without this test, an attacker who could control `CustomLogger.get_callback_env_vars()` could potentially use this path to exfiltrate arbitrary process-environment values into the config-API response.

## What I'd push back on (nits)

1. **Allowlist scope**: only generic-logger env vars are listed. If the project adopts more "read from env at runtime" callbacks later (e.g., a hypothetical `WEBHOOK_LOGGER_ENDPOINT`), each one is a new entry. Consider a more structural approach: have `CustomLogger` subclasses declare `env_var_source: Literal["config", "env"]` per var, so `process_callback` can route based on declared intent rather than a central allowlist. Not a merge blocker, but the allowlist will drift.
2. **`GENERIC_API_ENV_VARS` naming is slightly off**: the set is `{"GENERIC_LOGGER_..."}` but the constant is named `GENERIC_API_ENV_VARS`. Either rename the set to `GENERIC_LOGGER_ENV_VARS` (matches the keys) or rename the keys (don't). The current asymmetry will cause grep misses.
3. **The negative test is by far the most important** — recommend the PR body cite this explicitly so reviewers don't dismiss the test as boilerplate.
4. **No precedence test for "env set, config explicitly empty string"**. The current code returns `""` in that case (because `env_variable is None` is false for `""`). That's defensible but should be pinned with a test cell so future refactors don't flip it.
5. **`os.getenv(_var)` returns `None` on miss, same as the old behavior** — so a generic-logger env var that's unset everywhere still surfaces as `None`. Good. But worth a one-line comment at `:534` saying "explicitly preserve None on env miss, matching the non-allowlisted path" so a future contributor doesn't add a `or ""` default and silently change the API contract.

## Verdict

**merge-after-nits** — the fix is small, correctly diagnosed, and the four-test-cell coverage matrix (positive both callback names, precedence, negative-allowlist) is unusually principled for a proxy hygiene patch. The allowlist constant naming and missing "explicit empty string in config" cell are 5-minute fixes worth folding in.

Repo coverage: BerriAI/litellm (proxy config-API callback surfacing).
