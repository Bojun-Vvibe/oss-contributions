# BerriAI/litellm #26990 — chore(caching): isolate semantic cache entries

- URL: https://github.com/BerriAI/litellm/pull/26990
- Head SHA: `d8c11f962296feb8c7489bf6115062984e9716f7`
- Files: `litellm/caching/qdrant_semantic_cache.py` (+110/-68), `litellm/caching/redis_semantic_cache.py` (+103/-20), `litellm/proxy/_lazy_openapi_snapshot.{py,json}`, three test files (+535 net)

## Context / problem

VERIA-54: semantic-cache entries (Redis + Qdrant) were keyed by *embedding similarity alone* — there was no scope filter pinning entries to the LiteLLM-generated cache key. So a request with one set of params could be matched against a near-duplicate entry stored under a completely different scope (e.g. another tenant, another model, another `cache_control` policy). For caching that's nominally per-call-keyed but uses ANN lookup as the resolver, "no key filter" silently degrades to "global pool of semantically-similar responses." Cross-scope hit risk.

## Design analysis

The fix introduces a uniform `litellm_cache_key` payload field on every entry and requires every lookup to filter by it. Three coordinated pieces:

### 1. Field discipline (`qdrant_semantic_cache.py:18`, `redis_semantic_cache.py:316-319`)

Both backends define a single named constant:
```python
CACHE_KEY_FIELD_NAME = "litellm_cache_key"
```
All writes inject `self.CACHE_KEY_FIELD_NAME: str(key)` into the payload (`qdrant:137,231`; `redis:415,502` via `store_kwargs["filters"]`). All reads pass a key-scoped filter. The single named constant is the right pattern — no string-typo drift between the writer and the reader.

### 2. Lookup-side enforcement (defense in depth)

For Qdrant, `_get_qdrant_cache_key_filter(key)` (`:43`) builds the Qdrant `must` filter, AND the `_payload_matches_cache_key` post-check at `:74-78,:166,:290`:
```python
def _payload_matches_cache_key(self, payload: dict, key: str) -> bool:
    cached_key = payload.get(self.CACHE_KEY_FIELD_NAME)
    return cached_key is not None and str(cached_key) == str(key)
```
Belt + suspenders: even if the server-side filter were bypassed (older Qdrant version doesn't honor the filter, or the payload index hasn't been built yet), the client-side post-check rejects the hit. Comment at `:77` is explicit: *"…cannot be safely migrated to a generated LiteLLM cache key … so they must be treated as misses."* — fail-closed for unscoped legacy entries. Correct stance: a hit that *might* be cross-scope is treated as a miss, the user pays one extra LLM call, no leak.

For Redis, `_cache_hit_matches_key` at `:393-395` does the same post-check, and the lookup itself sets `filter_expression=self._get_cache_key_filter_expression(key)` (a `redisvl` `Tag(field) == str(key)` predicate at `:388-391`). Both layers gate.

### 3. Non-disruptive Redis index-schema upgrade (`redis_semantic_cache.py:357-384`)

The interesting one. An existing deployment has a `RedisSemanticCache` index *without* the `litellm_cache_key` filterable field. Re-creating the index would either drop existing entries or fail with a schema-mismatch `ValueError`. The fix at `:357-384`:
```python
try:
    return semantic_cache_cls(name=index_name, ..., filterable_fields=[self.CACHE_KEY_FILTERABLE_FIELD], overwrite=False)
except ValueError as exc:
    if "schema does not match" not in str(exc):
        raise
    isolated_index_name = f"{index_name}_isolated"
    print_verbose("Redis semantic-cache existing index schema is not isolated; using isolated index - {isolated_index_name}")
    return semantic_cache_cls(name=isolated_index_name, ..., filterable_fields=[self.CACHE_KEY_FILTERABLE_FIELD], overwrite=False)
```
Rather than mutate the legacy index in place (risk: drops data, can't roll back), a *new* index `<name>_isolated` is provisioned with the new schema, and the old one is left untouched. Old entries don't get cross-scope-leaked because they're in the unrestricted index that the new code never reads from. Combined with the dispositive test at `test_redis_semantic_cache.py:1208` (`test_redis_semantic_cache_uses_isolated_index_for_old_schema`) that injects `ValueError("Existing index schema does not match")` on the first init and asserts the isolated index is used, this is a textbook fail-safe migration shape.

Substring matching on `"schema does not match"` is the brittle part — if redisvl ever changes that error string (or localizes it), the `if "schema does not match" not in str(exc): raise` arm flips closed and re-raises. Worth widening: catch both `redisvl.exceptions.SchemaValidationError` (if such a typed exception exists) AND the substring fallback, or pin a redisvl version range.

### 4. Qdrant payload index (`qdrant_semantic_cache.py:53-65`)

`_ensure_cache_key_payload_index()` does best-effort `PUT /collections/{c}/index` with `field_name: "litellm_cache_key", field_schema: "keyword"`. Without the payload index, Qdrant will still apply the filter — it just walks the whole payload set rather than seeking — so this is performance-only, not correctness. Calling it from the `__init__` (`:27`) AND from the first lookup (`:35`) is belt + suspenders: an `__init__` failure (e.g. Qdrant unreachable at boot) doesn't leave the index unbuilt forever.

## Risks

- **Brittle substring match on `"schema does not match"`** (Redis path) — flagged above. If redisvl localizes or restructures that exception, the migration falls through to a hard raise instead of the fail-safe isolated-index branch. Pin the test to a specific redisvl version (the validation block already pins `redisvl==0.4.1`) and add a dependency-bump CI check that re-runs `test_redis_semantic_cache_uses_isolated_index_for_old_schema` against any newer redisvl.
- **No back-fill / migration tool for old-schema entries** — by design this PR doesn't migrate; old entries stay in the un-isolated index and are never read. That's the right scope (don't conflate isolation with migration), but the PR body should say so explicitly: "Operators with existing semantic caches will see those entries effectively orphaned — they continue to occupy Redis memory until manually flushed." Add an operator-doc note pointing to the `<name>_isolated` rename pattern.
- **`str(key)` coercion at write/read** — `self.CACHE_KEY_FIELD_NAME: str(key)` and `str(cached_key) == str(key)` both coerce. If the upstream `key` generator ever changes (different stringification for, say, a tuple-keyed namespace), the match silently breaks and *every* lookup becomes a miss. Worth a typed `CacheKey = NewType("CacheKey", str)` at the boundary so type-checkers catch a contract change.
- **Lazy OpenAPI snapshot churn** — the `_lazy_openapi_snapshot.json` and `.py` changes (PR body: "Stabilize lazy OpenAPI snapshot operation IDs so CI verification is deterministic") are bundled in but unrelated to the semantic-cache fix. Should be its own commit on the PR for cleaner bisect, but at this scale not worth blocking.

## Suggestions

- Switch the schema-mismatch detection to `isinstance(exc, SchemaValidationError) or "schema does not match" in str(exc)` if the typed exception exists.
- Add a negative test for Qdrant: write entry under key A, lookup with key B but identical embedding, assert miss (proves the post-check at `_payload_matches_cache_key` actually fires when the server-side filter is bypassed — e.g. by mocking the search to return the cross-scope hit).
- PR body should call out that this is correctness-positive but is a *behavior change for operators*: cache hit-rate will drop on first deployment (old entries become orphaned). Reasonable trade — cross-scope leakage was the bug — but operators monitoring hit-rate as an SLI need the heads-up.
- Document the `<name>_isolated` index name pattern in the operator docs so the next on-call who sees a doubled Redis memory footprint understands what they're looking at.

## Verdict

`merge-after-nits` — closes a real cross-scope-hit risk in semantic caching with the right architecture: a single named field constant (no typo drift), defense-in-depth lookup gates (server-side filter + client-side post-check), fail-closed treatment of unscoped legacy entries, fail-safe non-disruptive Redis index migration via `<name>_isolated` with a dispositive `test_redis_semantic_cache_uses_isolated_index_for_old_schema` regression, and best-effort Qdrant payload index built from both `__init__` and first lookup. Wants the substring-match brittleness widened, a typed `CacheKey` newtype, the Qdrant negative test, and operator docs for the orphaned-old-entries behavior change.
