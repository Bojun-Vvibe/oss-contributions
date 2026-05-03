# Review: BerriAI/litellm #27082 — fix(vector_store): resolve embedding config at request time

- **Repo**: BerriAI/litellm
- **PR**: #27082
- **Head SHA**: `74b4eab364348aa8cb22e4e15060ba78e083bd42`
- **Author**: stuxf

## What it does

Stops persisting resolved embedding-model credentials (`api_key`,
`api_base`, `api_version`) into the vector-store row at creation time.
Instead, the row stores only a model reference, and credentials are
resolved at request handling time via `_resolve_embedding_config()`,
backed by a short-TTL in-memory cache. Legacy rows that still carry a
resolved config pass through unchanged.

## Diff notes

- `litellm/proxy/vector_store_endpoints/endpoints.py:62-83` — at
  request time, if `litellm_embedding_model` is set but
  `litellm_embedding_config` is not, looks up the resolved config and
  builds a fresh `litellm_params` dict via spread. The inline comment
  correctly identifies the trap: the registry returns a *shared
  reference*, so an in-place `.update()` would persist the resolved
  cleartext into the in-memory registry cache for the process lifetime.
  Spread-to-fresh-dict avoids that. Nice catch.
- `litellm/proxy/vector_store_endpoints/management_endpoints.py:48-69` —
  introduces `_EMBEDDING_CONFIG_CACHE_TTL = 60`,
  `_EMBEDDING_CONFIG_CACHE_MAX_SIZE = 256`, and an `InMemoryCache`
  lazily allocated via `_get_embedding_config_cache()`. Lazy init is
  fine; the global is only mutated inside the helper.
- `_resolve_embedding_config()` (line ~314+) — checks the cache before
  hitting the router or DB, populates on hit. The comment explicitly
  states **negative results are not cached** so a freshly-added model
  isn't blocked behind the TTL. Correct call.

## Concerns

1. **TTL invalidation on credential rotation**: If an operator rotates
   the underlying model's API key in the router config or DB, the cache
   will keep serving the stale credential for up to 60s. For a security
   fix PR, this needs a paragraph in the PR description acknowledging
   the trade-off, plus probably a way to bust the cache on
   model-update events. A `_embedding_config_cache.flush()` call from
   the model-CRUD endpoints would be a 5-line follow-up.
2. **Cache key is the raw `embedding_model` string** — fine, but if two
   different teams have different overrides for the same model name
   (via `model_group_alias` or similar), they would collide. Worth
   confirming with the maintainers that embedding-model namespaces are
   global at this layer.
3. **Test coverage**: The PR description should call out which existing
   test suite covers (a) the fresh-row path (lookup happens), (b) the
   legacy-row path (lookup skipped), (c) the cache hit path. If no
   coverage exists for the cache invalidation behavior, add some.
4. The "never persist creds" claim is the headline. Verify the row-write
   path on the create/update endpoints actually drops
   `litellm_embedding_config` before persisting — that's not in the
   visible diff. If a legacy row still gets created with cleartext, the
   fix is half-done.

## Verdict

merge-after-nits
