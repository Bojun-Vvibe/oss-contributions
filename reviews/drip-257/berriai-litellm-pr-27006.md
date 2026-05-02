# BerriAI/litellm PR #27006 — fix(proxy): respect store_model_in_db=False during periodic DB model sync

- URL: https://github.com/BerriAI/litellm/pull/27006
- Head SHA: `84c3e6e8f59396a10fc149ed0ab4c24050957245`
- Author: AriOliv
- Verdict: **merge-as-is**

## Summary

Closes a footgun in `ProxyConfig.add_deployment` where the periodic background sync would pull models out of `LiteLLM_ProxyModelTable` and overwrite the in-memory router even when the operator had explicitly set `store_model_in_db=False`. A stray UI-added row could silently replace `config.yaml`-declared deployments on every sync. Fix gates the DB read on the documented `store_model_in_db` flag (general_settings or module global) *in addition to* the existing `_should_load_db_object("models")` check.

## Line-level observations

- `litellm/proxy/proxy_server.py` line 4967: `store_model_in_db` added to the `global` declaration — necessary because the new check reads the module-level binding (which is what the env-var path `STORE_MODEL_IN_DB=True` populates).
- `litellm/proxy/proxy_server.py` lines 4983–4997: the new gate is `general_settings.get("store_model_in_db", False) is True or store_model_in_db`. This correctly OR-combines (a) explicit `general_settings` from `config.yaml` and (b) the module global set via env var. Matches the precedent established elsewhere in the file for this flag and is consistent with the documented semantics ("Allow saving / managing models in DB"). The inline comment is unusually thorough for this codebase — keep it; it documents *why* the change is safe.
- The behavior change is conservative: when `store_model_in_db=False` (which is the default), no DB read happens at all, so `_get_models_from_db` and `_update_llm_router` are skipped. That is exactly what the test file asserts.
- `tests/test_litellm/proxy/test_add_deployment_store_model_in_db.py` — three tests cover the three meaningful states: (1) flag false → no DB read; (2) general_settings true → DB read; (3) module global true via env path → DB read. The patches use `unittest.mock.patch` against the live module path, which is the right approach for this codebase. Good coverage.
- Risk: any user who is *currently* relying on the buggy behavior (DB sync happening even with `store_model_in_db=False`) will see their UI-added models stop being applied at sync time. But since the docs already say "DB is not source of truth when this is False," that reliance was on undocumented behavior. Acceptable.

## Suggestions

1. Add a one-line entry to the changelog noting the fixed regression and pointing to the documented semantics, so anyone who happens to depend on the old behavior gets a clear migration note.
2. The third test (`test_loads_db_models_when_module_global_true`) is the subtle one — worth an inline docstring sentence explaining "this exercises the env-var path" so future readers don't merge it with test 2.
3. Optional: consider logging a `verbose_proxy_logger.debug(...)` line on the skipped path so operators tailing logs can see "model DB sync skipped because store_model_in_db is False" — would make the silent skip self-documenting in production.
