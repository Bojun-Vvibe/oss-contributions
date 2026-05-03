# Review — BerriAI/litellm #27082: fix(vector_store): resolve embedding config at request time, never persist creds

- **PR:** https://github.com/BerriAI/litellm/pull/27082
- **Head SHA:** `74b4eab364348aa8cb22e4e15060ba78e083bd42`
- **Author:** stuxf
- **Files:** 3 changed (+265 / −53)
  - `litellm/proxy/vector_store_endpoints/management_endpoints.py` (+62/−30)
  - `litellm/proxy/vector_store_endpoints/endpoints.py` (+27)
  - `tests/test_litellm/proxy/vector_store_endpoints/test_vector_store_endpoints.py` (+176/−23)

## Verdict: `merge-as-is`

## Rationale

This is a real security fix. Previously, `create_vector_store_in_db` and `update_vector_store` would call `_resolve_embedding_config(...)` at row-creation time and persist the resolved cleartext `api_key` / `api_base` / `api_version` into the row's `litellm_params.litellm_embedding_config` column (visible in the diff at `management_endpoints.py:469-481` of the *removed* code). Those credentials would then leak through every `vector_store` info/list/update response. The fix moves resolution to request-handling time in `endpoints.py:62-85` (`_update_request_data_with_litellm_managed_vector_store_registry`), where the resolved config lives only in a per-request `data` dict and is discarded when the call ends. The DB row keeps only the user-supplied `litellm_embedding_model` reference. That's the right architectural shape: cleartext creds never reach disk, never reach response bodies.

Two details are particularly well-handled. First, `endpoints.py:75-82` rebuilds `litellm_params` via `{**litellm_params, "litellm_embedding_config": resolved_config}` rather than mutating the registry's cached object — the comment in the diff calls out the trap explicitly ("the registry hands back a reference to its cached object, so an in-place update would persist the resolved cleartext into the in-memory cache for the lifetime of the process"). That is the correct fix; an in-place mutation would have re-introduced the leak through a different surface. Second, the new `_get_embedding_config_cache()` (TTL 60s, 256-entry LRU; `management_endpoints.py:48-69`) intentionally does *not* cache negative lookups — comment at `management_endpoints.py:329-333` — so a freshly-added model isn't blocked behind the TTL.

Backward compatibility for legacy rows that already carry a resolved cleartext config is handled correctly: `endpoints.py:71` checks `if embedding_model and not litellm_params.get("litellm_embedding_config")` so legacy rows skip the lookup and pass through unchanged, which keeps existing deployments working. The new redaction in `new_vector_store` at `management_endpoints.py:566-575` (`_redact_sensitive_litellm_params(...)`) closes the create-response leak as well.

The test file grows by +176/−23 — that's healthy coverage for a security-sensitive change. I'd merge this as-is and backport.

## Banned-string check

Diff scanned; no banned tokens present.
