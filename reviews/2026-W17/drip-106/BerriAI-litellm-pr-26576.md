# BerriAI/litellm PR #26576 — fix(proxy): load REDIS_* env vars when cache_params has non-connection keys

- **Link**: https://github.com/BerriAI/litellm/pull/26576
- **Head SHA**: `4eb1bf14f1e27ca05fe03116141345800c6139c4`
- **Author**: dschulmeist
- **Size**: +114 / -3 across 2 files
- **Verdict**: `merge-as-is`

## Files changed
- `litellm/proxy/proxy_server.py:3242-3252` — changes the gating condition for the REDIS_* env-var fallback in `ProxyConfig.load_config`. Old: `if (cache_type == "redis" or cache_type == "redis-semantic") and len(cache_params.keys()) == 0`. New: `if (cache_type == "redis" or cache_type == "redis-semantic") and ("host" not in cache_params and "url" not in cache_params)`. Adds a 4-line comment explaining the regression: "Gating on empty cache_params silently dropped to in-memory cache when unrelated keys like `mode` were set."
- `tests/test_litellm/proxy/test_proxy_server.py` — net +107 lines, two new async tests: `test_redis_env_vars_loaded_when_cache_params_has_non_connection_keys` (positive: with `mode: default_off` and REDIS_* env vars, env vars are honored) and `test_explicit_cache_params_host_not_overwritten_by_env_vars` (negative: explicit `host: explicit-redis.internal` wins over `REDIS_HOST: env-redis.internal`).

## Analysis

This is a 3-line behavior fix with disproportionately high operator value. The bug is exactly the kind of thing that bites a multi-pod proxy deployment in production weeks after deployment:

1. Operator sets `litellm_settings.cache: true` and exports `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD` env vars in their pod spec.
2. The proxy works correctly: cache type defaults to redis, env vars are honored, multi-pod spend tracking shares state via Redis.
3. Operator later adds `cache_params: {mode: default_off}` to the YAML to flip the default-off behavior (a feature added in some prior PR — orthogonal to connection details).
4. **Silent regression**: now `len(cache_params.keys()) == 1`, the `len(...) == 0` guard fails, the env-var fallback branch is skipped, and the proxy silently drops to `InMemoryCache`. Each pod now has its own isolated cache. Spend counters drift, rate-limit buckets reset per-pod, prompt-cache hit-rate plummets. No log line warns about this — `cache_type == "redis"` is still set in the resolved config, the operator sees their config "honored," and the only signal is the eventual cross-pod spend-counter desync.

The fix is the right gate: the predicate should be **"did the user supply connection details?"**, not **"did the user supply ANY cache_params?"**. The new check `"host" not in cache_params and "url" not in cache_params` is exactly that — connection-only. Both `host` (port/db/password assembly) and `url` (single-string redis URL) are the two ways to specify a Redis connection in the litellm config schema; if neither is present, env-var fallback is the right behavior.

The two-test coverage is excellent and shaped right:
- **Positive case** (`test_redis_env_vars_loaded_...`): explicitly sets `mode: default_off` plus the three REDIS_* env vars and asserts that the resolved `cache_params` from `ProxyConfig._init_cache(cache_params=...)` contains the env-var-derived `host`, `port`, `password` AND the user-supplied `mode`. The "Preserved user-supplied non-connection keys" assertion at the end is the load-bearing one — it pins that the env-var fallback merges, doesn't replace.
- **Negative case** (`test_explicit_cache_params_host_not_overwritten_by_env_vars`): explicit `host: explicit-redis.internal` plus `REDIS_HOST: env-redis.internal` env var. Asserts the explicit value wins. This is the *other* failure mode the new gate could plausibly introduce ("now env vars overwrite explicit config"), and the test pins that it doesn't.

The use of `monkeypatch.setenv` and `tempfile.NamedTemporaryFile(mode="w", suffix=".yaml", delete=False)` (with `os.unlink` in `finally`) is the right test-isolation pattern: env vars are per-test, and the YAML config file lives long enough for `load_config` to read it but is reliably cleaned up.

## Why merge-as-is

1. The fix is minimal (3 lines of behavior, 4 lines of comment).
2. The comment in the source explains *why* the old gate was wrong in operator-meaningful terms — it'll prevent the regression from being re-introduced by a future "clean up the conditional" PR.
3. Tests cover both the positive (env-var fallback fires) and negative (explicit config wins) cases.
4. The change is provably backward-compatible: any config that worked before either had `len(cache_params) == 0` (still works — neither `host` nor `url` is present) or had `host`/`url` set explicitly (still works — env vars don't overwrite). The only configs whose behavior changes are the ones that were *broken* before.
5. The fake-test-password value `"fake-test-password"` is clearly a literal, not a real secret.

## Optional follow-ups (not blocking)

- The two `cache_type == "redis" or cache_type == "redis-semantic"` checks would be more idiomatic as `cache_type in ("redis", "redis-semantic")` — minor.
- A third config-shape that could exhibit similar drift: `cache_params: {namespace: "my-app"}`. The new gate handles it correctly (no `host`/`url`, env vars fire), but a one-line additional test asserting that case would lock the contract for the next non-connection key someone adds.
- Worth a quick `rg 'len\(cache_params' litellm/` and `rg 'len\(.*\.keys\(\)\) == 0' litellm/proxy/` to see if the same anti-pattern exists elsewhere — checking for empty params as a proxy for "user didn't configure" is a common drift point.
- The PR description references "Re-opening previously closed PR #26244 against the new OSS staging branch" — the merge-train operator should confirm whether the original PR's review thread carried any outstanding feedback that needs to be addressed here.

## What I learned

The pattern "use `len(config.keys()) == 0` as a proxy for 'user didn't configure this thing'" is a very common bug. The right predicate is almost always "user didn't configure this *specific* thing", which means the gate should name what it's actually checking. The original code's `len() == 0` gate worked for years because nobody was setting non-connection keys; the moment a single non-connection key (`mode`) was added to the schema, the gate semantics quietly inverted. The fix is small but the lesson is general: gating predicates should always express their actual contract, not a heuristic that happened to be true at the time.
