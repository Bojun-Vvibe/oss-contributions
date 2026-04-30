# BerriAI/litellm #26867 — fix(proxy): coerce numeric budget settings from env var to float

- **Author:** xr843 (Tim Ren)
- **SHA:** `8272280`
- **State:** OPEN
- **Size:** +107 / -1 across `litellm/proxy/proxy_server.py` (+14) and `tests/test_litellm/proxy/test_max_budget_env_var.py` (+93)
- **Verdict:** `merge-as-is`

## Summary

Closes a config-loader type-coercion gap. When `litellm_settings.max_budget`
is sourced via `os.environ/MAX_BUDGET` in `config.yaml`, the env-var resolver
returns a `str` and the `litellm_settings` loop in `load_config` falls
through to the generic `setattr(litellm, key, value)` branch, leaving
`litellm.max_budget` as a string. The downstream startup check at
`proxy_server.py:924` (`prisma_client is not None and litellm.max_budget > 0`)
then raises `TypeError: '>' not supported between instances of 'str' and 'int'`.

Fix at `proxy_server.py:3329-3342` adds explicit `elif key == "max_budget"`,
`elif key == "max_user_budget"`, `elif key == "max_end_user_budget"`
branches that coerce via `float(value)` (with `None` preserved for the two
`Optional[float]` settings). The earlier fix #23843/#23855 only covered the
CLI path (`initialize(max_budget=...)`) and missed config.yaml.

## Reasoning

This is a clean, targeted fix in a high-traffic proxy startup path. Three
specific quality wins:

1. **The regression test `test_max_budget_from_config_yaml_env_var` at
   `tests/test_litellm/proxy/test_max_budget_env_var.py:54-94` actually
   exercises the failing path** — constructs a `tmp_path` config.yaml with
   `os.environ/TEST_MAX_BUDGET_REGRESSION`, sets the env var, runs
   `ProxyConfig.load_config`, and asserts both `isinstance(litellm.max_budget,
   float)` and the previously-failing `litellm.max_budget > 0` comparison.
   That's the right shape — the test will fail without the fix (PR body
   confirms "got `AssertionError: ... got str`").

2. **Symmetric defects bundled correctly.** The author noticed that
   `max_user_budget` and `max_end_user_budget` are both `Optional[float]` in
   `litellm/__init__.py` and both compared numerically downstream
   (`litellm/proxy/utils.py`). Same failure mode, same fix shape, same test
   pattern via `pytest.parametrize` at `:96-148`. Fixing one and leaving the
   other two as latent bugs would be worse OSS hygiene.

3. **The `None`-preservation discipline is correct** — `max_budget` defaults
   to `0.0` (int comparison `> 0` works on either, but the existing default
   in `litellm/__init__.py` is float so the coercion preserves type
   consistency), while `max_user_budget` and `max_end_user_budget` preserve
   `None` so the downstream "is the cap configured?" predicate
   (`if litellm.max_user_budget is not None`) keeps working.

The PR body's "empty/whitespace env var handling not added here; matches the
behavior of the existing CLI-path coercion at `proxy_server.py:5885` which
is also a bare `float()`" is the right call — preserving parity with the
existing behavior so `LITELLM_MAX_BUDGET=""` produces the same error in both
code paths is better than silently fixing one and not the other. A follow-up
hardening PR adding `try: float(value.strip()) except ValueError: raise
ConfigError(...)` to both sites is the right shape if anyone wants to chase
it.

The test_plan output (5/5 passing including the regression and the parametrized
case) and the fact that this is the explicit closure of #26696 make this a
clean merge.
