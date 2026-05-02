# Review — BerriAI/litellm#26990

- PR: https://github.com/BerriAI/litellm/pull/26990
- Title: chore(caching): isolate semantic cache entries
- Head SHA: `8cb52ce0bbcae4d1e421796224543a4c118fa41f`
- Size: +765 / −125 across 4 files
- Verdict: **merge-after-nits**

## Summary

Hardens both `QdrantSemanticCache` and `RedisSemanticCache` so a
semantic-similarity hit only returns when the cached entry was stored
under the *same* generated cache key as the lookup. Previously,
embeddings stored from one tenant/key could be returned to a totally
different caller with a similar prompt — this PR plugs that.

## Evidence / specific spots

- `litellm/caching/qdrant_semantic_cache.py`:
  - New class constant `CACHE_KEY_FIELD_NAME = "litellm_cache_key"`
    (line 27) used as the payload field.
  - `_ensure_cache_key_payload_index()` (lines 188-209) issues a
    `PUT /collections/{name}/index` with `field_schema: keyword` so
    Qdrant can filter on the new field.
  - `_payload_matches_cache_key()` (lines 211-216) explicitly returns
    `False` for legacy points that lack the field — comment
    correctly notes "they cannot be reassigned to the generated
    LiteLLM cache key without risking cross-scope hits".
  - `set_cache()` now writes `self.CACHE_KEY_FIELD_NAME: str(key)`
    into the payload alongside `text`/`response`.
  - Prompt assembly switched from manual `for message in messages:
    prompt += message["content"]` to `get_str_from_messages(messages)`,
    which matches what the rest of the proxy uses and avoids
    `TypeError` on non-string content blocks.
- `litellm/caching/redis_semantic_cache.py` (+123) — same pattern via
  Redis hash fields.
- New test files cover both caches (~530 lines total of test code).

## Notes / nits

- The legacy-point handling silently treats orphans as a miss but
  doesn't evict them. Over time a busy Qdrant collection will accumulate
  unreachable points that consume disk + slow vector search. A
  follow-up should either add a one-shot migration script or a TTL on
  legacy points.
- `_ensure_cache_key_payload_index` swallows non-2xx as a `print_verbose`
  warning and continues. If the index creation actually fails, every
  subsequent filtered query will be a full scan (or error, depending on
  Qdrant version). Worth surfacing to `verbose_proxy_logger.warning` so
  ops can see it instead of buried verbose output.
- The dynamic import of `proxy_server.llm_router` in
  `_get_async_embedding` is duplicated logic — there's already a similar
  helper elsewhere in the cache module surface; would be cleaner as a
  shared `_get_proxy_router()` util.
- Fallback path (no `llm_router`) calls `litellm.aembedding` with
  `cache={"no-store": True, "no-cache": True}` — good (prevents
  recursive cache-of-the-cache), matches the routed branch.

Solid security/correctness fix. Address the index-creation logging and
land it.
