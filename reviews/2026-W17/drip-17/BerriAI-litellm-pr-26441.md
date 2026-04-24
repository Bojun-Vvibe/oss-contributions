# BerriAI/litellm PR #26441 — fix(redis): cache GCP IAM token to prevent async event loop blocking

- **Author:** harish-berri
- **Head SHA:** 7021eb41c7e6f33afdbff93770bab533fd754e3b
- **Files:** `litellm/_redis_credential_provider.py`,
  `tests/test_litellm/test_redis.py` (+133 / −16)
- **Verdict:** `merge-as-is`

## Context

The previous `GCPIAMCredentialProvider` minted a fresh IAM token on
every `get_credentials()` / `get_credentials_async()` call. That was
itself a fix for an earlier bug where a stale-at-startup token was
cached statically and broke after the 1h IAM expiry. But the
"fix" overshot: every new Redis connection (pool warm-up, health
checks, sentinel reconnects) now triggers a synchronous
google-auth network round-trip. Under cluster warm-up that
serialises inside Python's GIL/event loop and produces cascading
request latency.

## What the diff does

In `litellm/_redis_credential_provider.py`:

- Lines 7–14 introduce a module-level
  `_token_cache: Dict[str, Tuple[str, float]]` keyed by service
  account, plus a `threading.Lock`. TTL is 3300s (55 min) — a
  conservative refresh window inside the 1h IAM lifetime.
- Lines 45–80 add `_get_cached_gcp_iam_token` with double-checked
  locking: fast-path read outside the lock, re-check inside the
  lock, then mint+store. Returns the cached token whenever
  `time.monotonic() < expiry`.
- Lines 92, 96 swap the two callers — sync `get_credentials()` and
  async `get_credentials_async()` (still wrapped in
  `asyncio.to_thread` to keep the blocking google-auth call off the
  event loop) — to call `_get_cached_gcp_iam_token` instead of
  `_generate_gcp_iam_access_token`.

Tests at `tests/test_litellm/test_redis.py:23-29` add an
autouse fixture that clears `_token_cache` between tests
(critical — without this, test ordering would silently couple
cases). New tests:

- `test_gcp_iam_credential_provider_caches_token` (line 218):
  5 calls, 1 mint.
- `test_gcp_iam_credential_provider_refreshes_on_expiry` (line 245):
  manually backdates the cache entry, asserts a second mint.
- `test_gcp_iam_credential_provider_cache_shared_across_instances`
  (line 273): two provider instances for the same SA share the
  cached token.

## Review notes

- The double-checked locking is implemented correctly: the
  outer read is lock-free, and the inner read inside the
  `with _token_cache_lock:` block re-validates after acquiring
  the lock. `time.monotonic()` is the right clock here (immune
  to wall-clock jumps).
- The 55-min TTL leaves a 5-min safety window before the 60-min
  IAM expiry. Reasonable. If a request grabs a token at minute
  54.9 and the connection takes 60s to authenticate, the token
  could still expire mid-flight — but that's the IAM provider's
  problem, not this cache's. The shorter TTL is a fine choice.
- Cache key is the service account string. Correct for the
  single-tenant case. Multi-tenant deployments that rotate
  service accounts via `_gcp_service_account` mutation would
  also work, since each new SA value seeds a new cache entry.
  Old entries leak forever, but the cardinality is tiny in
  practice.
- The `asyncio.to_thread` wrapper at line 96 is preserved
  unnecessarily for the cached path — when `_token_cache` hits,
  `_get_cached_gcp_iam_token` returns in microseconds and doesn't
  need a thread hop. Stripping it would cost an `if cached: return
  ...` branch on the hot path, which isn't worth the complexity.
  Leave as is.
- Test coverage is thorough: hit, miss-after-expiry, shared-instance.
  Missing only a concurrent-access test (e.g. 10 threads racing
  `get_credentials()` for the same SA, asserting `mock_gen.call_count == 1`).
  That would prove the lock actually works under contention. Not
  blocking; the logic is short enough to inspect.
- The autouse `clear_gcp_iam_token_cache` fixture is the right
  call. Module-level caches and pytest are a notorious flake
  source.

## What I learned

The progression — static-cache (broken on expiry) → no-cache
(broken under load) → TTL-cache (correct) — is the canonical
authentication-token-caching arc. Worth noting the implementation
detail: `time.monotonic()` over `time.time()` for any TTL that
must survive NTP adjustments or DST, and module-level `Dict` +
`threading.Lock` over an LRU when the cardinality is bounded by
the number of distinct service accounts (typically 1).
