# BerriAI/litellm#26790 — fix(redis_cache): apply namespace prefix in delete_cache and async_delete_cache

- **PR:** https://github.com/BerriAI/litellm/pull/26790
- **Author:** @elluvium
- **Head SHA:** `62473fc4` (full: `62473fc4ffc54e53b3f85b757f1685ac29713d3c`)
- **Size:** +29/-3 across 2 files

## What this does

Fixes a real and embarrassing parity bug in `litellm/caching/redis_cache.py`:
`async_delete_cache` and `delete_cache` were the only public cache methods
that did not call `self.check_and_fix_namespace(key=key)` before issuing the
Redis command. So when a `namespace` was configured (very common in proxy
deployments), `SET`/`GET` correctly wrote/read at e.g.
`litellm.caching.caching:cronjob_lock:db_spend_update_job`, but `DEL` was
issued against the raw key (`cronjob_lock:db_spend_update_job`) — which
does not exist — and always returned `0`.

The downstream visible symptom called out in the PR description matches
what I'd expect from this primitive: `PodLockManager` could never explicitly
release locks; every lock had to expire by TTL, and every batch-write cycle
on every pod logged `failed to release Redis lock`.

## The fix

Two-line change in `redis_cache.py:1262-1265`:

```python
async def async_delete_cache(self, key: str):
    _redis_client: Any = self.init_async_client()
-   # keys is str
+   key = self.check_and_fix_namespace(key=key)
    return await _redis_client.delete(key)

-def delete_cache(self, key):
+def delete_cache(self, key: str):
+   key = self.check_and_fix_namespace(key=key)
    self.redis_client.delete(key)
```

(The sync version also gains the missing `key: str` type hint, which is a
nice drive-by.)

## Test coverage

`tests/test_litellm/caching/test_redis_cache.py:476-499` adds
`test_delete_cache_applies_namespace`, parametrized over
`namespace=None` and `namespace="myns"`, asserting both async and sync
`.delete()` are invoked with the namespaced key:

```python
expected_key = f"{namespace}:{raw_key}" if namespace else raw_key
…
mock_async_client.delete.assert_called_once_with(expected_key)
…
mock_sync_client.delete.assert_called_once_with(expected_key)
```

The `namespace=None` parametrization also locks in the contract that no
prefix is applied when no namespace is configured — i.e. it actively
prevents the inverse regression. Good test design.

## What's worth thinking about (post-merge, not blocking)

1. **Why was this missed in `delete_cache` specifically?** Other methods
   (`set_cache`, `get_cache`, `increment_cache`, etc.) all do the
   `check_and_fix_namespace` call. If there's a base class or pattern
   that's supposed to enforce this and `delete_cache` just got skipped
   when namespace support was added, a `RedisCacheBase` ABC with an
   abstract `_namespaced_op(key, fn)` helper would prevent the next
   parity bug. Not for this PR, just worth filing as a follow-up.

2. **Pipeline / multi-key delete paths.** This PR fixes the single-key
   `DEL`. There's a `_pipeline_increment_helper` immediately below the
   patched function; worth one `grep -n 'redis_client\.delete\|redis_client\.unlink' litellm/caching/`
   to confirm there isn't a sibling `bulk_delete` / `unlink` / pipeline
   `DEL` path with the same bug.

3. **The bug also implies a silent telemetry hole** — `PodLockManager`'s
   "lock released" counter (if any) was always wrong for namespaced
   deployments. After this lands, that counter will jump on every pod;
   worth a release-note callout so operators don't think something
   changed in lock acquisition.

## Verdict

**`merge-as-is`** — minimal, correct, parametrized regression test,
tight blast radius. The kind of fix that should land same-day.

## Nits

- None blocking. (Optional: rename `tests/.../test_delete_cache_applies_namespace`
  to `test_delete_cache_applies_namespace_prefix` for symmetry with how
  `check_and_fix_namespace` is described in the codebase, but really not
  worth a re-push.)
