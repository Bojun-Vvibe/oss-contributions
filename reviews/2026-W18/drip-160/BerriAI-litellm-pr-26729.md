# BerriAI/litellm PR #26729 — Add negative caching for invalid virtual keys

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26729
- Head SHA: `22b653a8934518c9f9d427e794730396122f1f8d`
- State: OPEN, +424/-26 across 7 files

## What it does

Closes a DB-amplification attack vector: previously every request bearing an invalid `sk-...` virtual key triggered the full `get_key_object` codepath, which queries `combined_view` (a relatively heavy multi-table join). An attacker (or a misconfigured client) hammering with the same bad key generated unbounded DB load. This PR introduces an `InvalidVirtualKeyCache` layer that pre-flights the auth path:

1. Validates `sk-` prefix shape (raises 401 on malformed shape directly — pulled out of the inline `assert` in `user_api_key_auth.py`).
2. Checks an in-memory negative cache keyed on `invalid_vk:{hashed_token}` (TTL: 3600s by default, configurable per-deployment).
3. If not cached, runs a *cheap* probe against the `litellm_verificationtoken` table (single-table, indexed on `token`); cache-misses get recorded with the configured TTL.

The matching invalidation hook fires on key creation in `generate_key_helper_fn` (line 3248) so a previously-bad-then-created key doesn't stay 401'd for an hour.

## Specific reads

- `litellm/proxy/auth/reject_invalid_tokens.py:225-256` — the core preflight:
  ```python
  hashed_token = hash_token(token=api_key)
  if ttl_seconds is None:
      return False
  if not await cls.allows_db_lookup(hashed_token=..., ttl_seconds=...):
      return True
  # ... probe verification token table ...
  if not token_probe_failed and token_row is None:
      await cls.record_miss(...)
      return True
  return False
  ```
  Cache shape is correct: only records a miss when the probe definitively says "not in DB" (`token_probe_failed and token_row is None`). On probe failure (line 240-246, `verbose_proxy_logger.debug` + `token_probe_failed = True`), the function falls through to "let combined_view try" — fail-open under DB error, which is the right semantic (don't escalate a transient DB blip into mass auth failures).

- `litellm/proxy/auth/reject_invalid_tokens.py:154-156` — the cache-read fail-open is also explicit:
  ```python
  except Exception as e:
      verbose_proxy_logger.debug("InvalidVirtualKeyCache.allows_db_lookup: cache read failed, allowing query: %s", e)
      return True
  ```
  Good. A flaky Redis here would only de-optimize, not break auth.

- `litellm/proxy/auth/user_api_key_auth.py:1167-1200` — the integration into `_user_api_key_auth_builder`:
  ```python
  if await InvalidVirtualKeyCache.check_invalid_token(...):
      raise ProxyException(
          message="Authentication Error at InvalidVirtualKeyCache, Invalid proxy server token passed. Token (hash) = {}. Unable to find token in cache or `LiteLLM_VerificationTokenTable`".format(hash_token(token=api_key)),
          ...
      )
  api_key = hash_token(token=api_key)
  ```
  **Concern**: The `api_key` is hashed *twice* on the negative-cache-hit branch — once inside `check_invalid_token` (`reject_invalid_tokens.py:223`), and again at line 1184 in `user_api_key_auth.py`. Hashing is bcrypt-light (`hash_token` is SHA-256-based per the import) so cost is small, but `hash_token(token=api_key)` in the error message also recomputes — three hashes per rejected request. A cleaner shape would have `check_invalid_token` return `(reject: bool, hashed: str)` so the hashed value is reused.

- `litellm/proxy/auth/reject_invalid_tokens.py:75-103` — the TTL resolution:
  ```python
  raw = general_settings.get("invalid_virtual_key_cache_ttl")
  if raw is None:
      litellm_settings = general_settings.get("litellm_settings")
      if isinstance(litellm_settings, dict):
          raw = litellm_settings.get("invalid_virtual_key_cache_ttl")
  default_ttl = float(DEFAULT_INVALID_VIRTUAL_KEY_NEGATIVE_CACHE_TTL_SECONDS)
  try:
      if raw is None: return default_ttl
      v = float(raw)
  except (TypeError, ValueError):
      return default_ttl
  if v <= 0:
      return None
  return v
  ```
  Two-tier resolution (top-level then nested `litellm_settings`) matches the surrounding config conventions. The `<= 0 → None → disabled` semantic is sensible ("set to 0 to turn off"). Default is 1 hour (`DEFAULT_INVALID_VIRTUAL_KEY_NEGATIVE_CACHE_TTL_SECONDS = 3600`) — aggressive enough to absorb a hammer attack, short enough that a legitimate key created shortly after probing won't be locked out for too long.

- `litellm/proxy/management_endpoints/key_management_endpoints.py:3248-3252` — the invalidation hook:
  ```python
  if key_data["token_id"] is not None:
      await InvalidVirtualKeyCache.delete_invalid_token_cache(
          hashed_token=key_data["token_id"],
          user_api_key_cache=user_api_key_cache,
      )
  ```
  Correct — invalidates the negative cache entry for the freshly-created key's hash. **But**: this only fires on `generate_key_helper_fn`. Other key-creation paths (key import via `/key/import`, key restoration after delete-undo, key updates that change the token value) are not hooked. A lingering 401 for up to 1 hour after restoring a previously-deleted key is a real possibility.

- Test files: `test_reject_invalid_tokens.py` (134 lines, 4 cases) covers the four core branches — empty-cache + DB-miss → record-miss + reject; cache-hit → reject without DB call; cache-delete → removes negative entry; empty-cache + DB-hit → preflight passes. Good. `test_key_generate_invalid_token_cache.py` (46 lines) pins the invalidation hook fires on key creation. **Missing**: tests for the malformed-key 401 path (was an `assert`, is now `raise HTTPException`) — behavioral change from `AssertionError` to a 401 means callers catching `AssertionError` upstream will silently break.

## Verdict: `merge-after-nits`

## Rationale

This is a high-value defensive fix to a real DB amplification surface, the structure (preflight → negative cache → cheap probe → fall-through to full auth) is exactly the right shape, and the fail-open semantics on cache/probe errors are correct (don't let observability infra take down auth). Three nits before merge: (a) `check_invalid_token` should return `(reject, hashed_token)` so the caller doesn't re-hash the same key 2-3 times per request; (b) the invalidation hook in `generate_key_helper_fn` is a good start but other key-creation/restoration paths (`/key/import`, undelete) need parallel invalidations or the cache will grant rejected-status to a freshly-restored valid key for up to TTL; and (c) the malformed-key path silently changed from `AssertionError` to `HTTPException(401)` — behavioral change deserves its own test pinning the 401 status and (optionally) a call-site audit for upstream `try: ... except AssertionError:` handlers that might silently break.
