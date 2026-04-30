# BerriAI/litellm PR #26851 — chore(proxy): block env callback refs in key metadata

- PR: https://github.com/BerriAI/litellm/pull/26851
- Head SHA: `1ebb192cbed0335835a5c62fba8b4ad158ff33e6`
- Files touched: `litellm/litellm_core_utils/initialize_dynamic_callback_params.py` (+13/-4), `litellm/proxy/_types.py` (+7/-2), `litellm/proxy/litellm_pre_call_utils.py` (+32/-5), `tests/proxy_unit_tests/test_proxy_utils.py` (+42/-10), `tests/test_litellm/proxy/test_litellm_pre_call_utils.py` (+72/-19)

## Specific citations

- New shared validator `validate_no_callback_env_reference` extracted at `initialize_dynamic_callback_params.py:26-31` so the same `_is_env_reference` → `_raise_env_reference_error` pattern is used at both request-body parse path (`:73-78`) and metadata-fallback path (`:88-93`).
- The load-bearing security fix at `proxy/_types.py:1871-1879` — `AddTeamCallback.validate_callback_vars` now coerces every value to `str` *and* invokes `validate_no_callback_env_reference(key, callback_vars[key], source="key/team callback metadata")` for every entry. Before this change, key-level metadata could carry `langfuse_secret_key: "os.environ/MY_VAR"` and the proxy resolved it server-side, escalating a key-scoped configuration into a server-environment dereference primitive.
- `litellm_pre_call_utils.py:228-232`: `convert_key_logging_metadata_to_callback` now stores `str(value)` directly without going through `litellm.utils.get_secret(value, default_value=value)` — closes the get_secret call site that was the bypass mechanism.
- New `_get_validated_callback_metadata` helper at `:240-251` wraps `AddTeamCallback(**item)` in `try/except (PydanticValidationError, ValueError)`, logs `"Ignoring invalid %s callback metadata: %s"` via `verbose_proxy_logger.warning(... _sanitize_for_log(str(e)))` and returns `None` so a malformed/disallowed entry causes the metadata to be skipped (returning `callbacks is None`) rather than raising up to the request handler.
- Two call sites updated at `:289-296` (key-level) and `:301-308` (team-level) to skip on `None` return.
- New helper applied at `:923-931` `add_team_based_callbacks_from_config`: `callback_vars_dict` is rebuilt with `litellm.utils.get_secret(value, default_value=value) or value` *only when value is a str* — this is the **legitimate** server-side env-resolution path (used by config.yaml-defined callbacks, which are admin-trusted).
- Negative regression at `test_proxy_utils.py:312-345`: `test_dynamic_logging_metadata_ignores_env_references_from_key_metadata` patches `litellm.utils.get_secret` to `pytest.fail("get_secret should not be called")` — proves the legitimate env-resolution path is *not* invoked for key-level metadata. Asserts `callbacks is None`.
- Pydantic-level regression at `test_litellm_pre_call_utils.py:1278-1289`: `test_add_team_callback_rejects_env_reference` asserts `AddTeamCallback(...)` raises `PydanticValidationError` with `"os.environ/"` in the message.
- Positive regression preserved at `test_proxy_utils.py:1290-1303` — admin-trusted config.yaml path still resolves `langfuse_public_key` / `langfuse_secret` through `get_secret`.

## Verdict: merge-as-is

## Concerns / nits

1. The failure mode is **silent skip with a warning log** (`_get_validated_callback_metadata` returns `None` → handler returns `callbacks is None`). This is the right safety choice (don't break the request) but operators with dashboards keying on log-line counts may want a metric counter. A `verbose_proxy_logger.warning` is the floor; a `DEPLOYMENT_CALLBACK_REJECTED` metric on top would let SREs detect attacker probing without log scraping.
2. The `_sanitize_for_log` wrap at `:248` is the right call — pydantic error messages echo the offending value, which would otherwise put the rejected `os.environ/...` reference (potentially attacker-injected) in operator logs.
3. The legitimate env-resolution carve-out at `:923-931` — `litellm.utils.get_secret(value, default_value=value) or value if isinstance(value, str) else value` — has a subtle truthiness trap: if `get_secret` returns `""` (env var explicitly set to empty), the `or value` fallback restores the env-reference *string*, which then ships to the callback as the literal `"os.environ/FOO"` not the resolved empty value. Probably an edge case but worth a docstring note.
4. `AddTeamCallback.validate_callback_vars` now coerces every value to `str` unconditionally at `:1875` (previously only on `not isinstance(value, str)`). Confirm no callback shape relies on a non-str value type (e.g. integer `langfuse_flush_at`) — the diff suggests the schema is str-only but worth a sanity grep.
5. The test rename / parametrize cleanup at `test_proxy_utils.py:235-244` — dropping the second parameterized case that *exercised* the now-blocked `os.environ/...` form — is the correct shape since the new test at `:312-345` covers the inverse assertion.
