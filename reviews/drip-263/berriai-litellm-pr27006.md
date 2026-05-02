# BerriAI/litellm PR #27006 — fix(proxy): respect `store_model_in_db=False` during periodic DB model sync

- **Head SHA**: `84c3e6e8f59396a10fc149ed0ab4c24050957245`
- **Scope**: `litellm/proxy/proxy_server.py` + new regression test

## Summary

`ProxyConfig.add_deployment` previously gated DB model loads only on `_should_load_db_object("models")` (default-on). With `store_model_in_db=False`, a stray UI-added row in `LiteLLM_ProxyModelTable` could overwrite `config.yaml` models on every periodic sync. Fix at `litellm/proxy/proxy_server.py:4983-4997` adds a precondition:

```python
store_model_in_db_enabled = (
    general_settings.get("store_model_in_db", False) is True
    or store_model_in_db
)
if store_model_in_db_enabled and self._should_load_db_object(object_type="models"):
    ...
```

## Comments

- Both the env-var path (module global `store_model_in_db`) and the `general_settings` path are honored — important because the env var sets only the module global when `general_settings` is silent.
- New tests in `tests/test_litellm/proxy/test_add_deployment_store_model_in_db.py` cover all three code paths: false-disables, true-via-settings, true-via-module-global. Solid regression net.
- Adding `store_model_in_db` to the `global` declaration at line 4967 is needed because the function now reads the module global; correct.
- One nit: the `is True` check on line 4992 is intentional (rejects truthy non-bool like `"True"` strings). That's defensible but worth a comment — silent type-strictness on config keys can surprise users who YAML-cast booleans.
- Comment block at lines 4983-4988 is excellent — captures the "why" precisely.

## Verdict

`merge-as-is` — clear root cause, narrow fix, good tests.
