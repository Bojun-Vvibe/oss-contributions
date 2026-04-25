# BerriAI/litellm #26524 — fix(vector_stores): restrict deletion to proxy admins

- URL: https://github.com/BerriAI/litellm/pull/26524
- Head SHA: `64daf17449589052eb3be4d6a11bdccd2a3af39f`
- State: OPEN
- Files: `litellm/proxy/vector_store_endpoints/endpoints.py`,
  `litellm/proxy/pass_through_endpoints/llm_passthrough_endpoints.py` (+27/-1)
- Verdict: **merge-after-nits**

## What it does

Closes a privilege gap where any valid API key — not just proxy admin
keys — could `DELETE /v1/vector_stores/{id}` and
`DELETE /azure_ai/indexes/{name}`. The OpenAI-shaped vector store route
relied on `user_api_key_auth` only (no role check), and the Azure AI
Search pass-through fell through to the generic Azure proxy when the
index was missing from the in-memory `allowed_vector_store_indexes`
registry — quietly forwarding the DELETE upstream.

## Diff notes

Three guarded sites:

1. `vector_store_endpoints/endpoints.py:501-507` — `vector_store_delete`
   now front-loads `_is_proxy_admin(user_api_key_dict)` and 403s with
   `"Only proxy admins can delete vector stores."` before any of the
   `general_settings`/`llm_router` imports run. Correct ordering — the
   check fires before any I/O.

2. `pass_through_endpoints/llm_passthrough_endpoints.py:1370-1378` —
   on the `is_vector_store_index` branch, a DELETE-method gate is
   added before the provider config lookup. Same 403 shape.

3. `pass_through_endpoints/llm_passthrough_endpoints.py:1437-1443` —
   the registry-miss tail path (the actual reported bypass) gets the
   same gate. Without this, an attacker with a non-admin key could
   pick an index name *not* registered in
   `allowed_vector_store_indexes` and the request would slip through
   to the Azure backend unchecked.

The new `_is_proxy_admin` import comes from the existing
`vector_store_endpoints/utils.py`; consistent helper, no new auth
surface introduced.

## Risk surface / nits

- **Nit**: the registry-miss check at line 1437 fires *after* the
  earlier `is_vector_store_index` branch already 403'd — defensible
  as defense-in-depth, but the 403 message and gate are duplicated
  verbatim. Worth extracting to a single guard at the top of the
  vector-store-shaped portion of `azure_proxy_route`, or at minimum
  adding a comment noting the layering is intentional.
- The OpenAI-shaped `vector_store_create`/`vector_store_modify` paths
  are not touched. PR title says "deletion" so that's in-scope, but
  the same auth gap is plausible for create/modify on Azure indexes
  (see `is_vector_store_index` branch above) and worth a follow-up.
- No regression test for the registry-miss tail path, which is
  precisely the case that motivated the PR. A unit test asserting
  `403` on a fake non-admin key + unregistered index name would lock
  down the bypass.
- `_is_proxy_admin` (single underscore) is treated as a public-ish
  helper across two import sites in this PR; if there's a non-private
  alias intended, this is the moment to rename.

## Why this verdict

Correct, narrowly-scoped security fix on a real privilege gap. Three
issues to address (regression test on the registry-miss case, layered-
guard comment or extraction, and a follow-up note about
create/modify), but the core change is sound and merge-blocking only
on the missing test.
