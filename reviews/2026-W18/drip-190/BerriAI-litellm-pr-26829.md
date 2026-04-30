# BerriAI/litellm #26829 — Refresh Redis TTL on counter writes, skip stale in-memory in Redis

- **URL:** https://github.com/BerriAI/litellm/pull/26829
- **Head SHA:** `86825286a1a5b574b7a58299b05cfb68feb2a448`
- **Files:** `litellm/caching/{dual_cache,redis_cache}.py`, `litellm/proxy/common_utils/reset_budget_job.py`, `litellm/proxy/db/spend_counter_reseed.py`, `litellm/proxy/proxy_server.py` + 3 test files (+382/-44)
- **Verdict:** `merge-after-nits`

## What changed

Two-bug fix on the spend-counter cache hierarchy:

1. **TTL never refreshed → Redis evicts a hot counter mid-billing-window.** `redis_cache.py:824-845` adds an opt-in `refresh_ttl: bool = False` param to `async_increment`. When `True`, every write calls `await _redis_client.expire(key, _used_ttl)` unconditionally; when `False` (default), the previous "set TTL only on key-creation" semantics are preserved (rate-limit windows still want fixed-window expiration). Wired through `dual_cache.py:392-426` (`async_increment_cache(refresh_ttl=...)`), then enabled at the three spend-counter write sites: `proxy_server.py:1975` (initial seed), `:1979-1981` (per-request increment), and `spend_counter_reseed.py:155` (reseed-from-DB warm).
2. **In-memory masking cross-pod increments.** `proxy_server.py:1790-1817` and `spend_counter_reseed.py:131-148` add a `redis_clean_miss = False` flag: if Redis returned `None` (key absent, not error), skip the in-memory cache check entirely and go straight to DB-reseed. Per-pod in-memory only contains *this* pod's writes, so on a clean Redis miss it would return a stale per-pod value that masks increments from other pods. The fall-through to in-memory is preserved when Redis is *unreachable* (transient error) — that case still legitimately wants per-pod in-memory as a fallback.

Plus an invalidation refactor at `reset_budget_job.py:55-77`: extracts `_invalidate_spend_counter(counter_key)` static helper that zeros both in-memory and Redis (`async_set_cache(value=0.0)`) with a `verbose_proxy_logger.warning` on Redis failure naming the over-enforcement consequence ("Budget may be over-enforced until counter expires"). Called from four reset paths: team members (`:88-104`), keys-linked-to-budgets (`:130-153`, with new pre-fetch `find_many` to enumerate keys), per-key budget reset (`:380-384`), per-user budget reset (`:466-472`), per-team budget reset (`:563-569`).

## Why it's right

- **Bug 1 diagnosis is canonical.** A spend counter at TTL=60s that gets incremented at second 50 should still be there at second 70, but the previous code never refreshed the TTL, so the counter expired at second 60 and the next request reseeded from DB — every reseed is a synchronization point that *cannot* see in-flight increments from other pods, opening a budget-bypass window. Refreshing TTL on every increment closes it.
- **Bug 2 diagnosis is canonical.** The whole point of using Redis is cross-pod consistency. The previous fall-through to in-memory on Redis-clean-miss meant that any pod that had ever served a request for a given counter would return its *local* view, even if other pods had since incremented in Redis (which then expired). Distinguishing "Redis miss" (go to DB-reseed, authoritative) from "Redis error" (graceful degradation to in-memory) is exactly the right semantic split, and the `redis_clean_miss` flag at `:1790, 1797, 1809-1812` is the cleanest expression.
- **Test contracts.**
  - `test_redis_cache_async_increment_refresh_ttl_true_bumps_existing_ttl` at `test_redis_cache.py:54-72` asserts `expire.assert_awaited_once_with(key, 60)` even when `ttl.return_value = 42` (key already has time left) — locks "always bump".
  - `test_redis_cache_async_increment_default_does_not_bump_existing_ttl` at `:75-93` asserts `expire.assert_not_awaited()` for the rate-limit window case — locks the default-`False` non-regression.
  - `test_reset_budget_for_team_members_invalidates_redis_counter` and `test_reset_budget_for_keys_invalidates_redis_counter` at `test_reset_budget_job.py:1057-1148+` lock the four new invalidation paths via `assert_any_await(key=..., value=0.0)` against the stub.
- **The `_invalidate_spend_counter` extraction** centralizes the "zero both layers + warn on Redis failure" pattern; previously it was inlined only in the team-member path and silently absent from the other four reset paths. Extracting + applying uniformly closes a real budget-over-enforcement gap on key/user/team resets.

## Nits / not blockers

- **Default `refresh_ttl=False` is a footgun for new callers.** Anyone adding a new spend-counter increment (or future operator-style accumulation) and forgetting `refresh_ttl=True` recreates Bug 1. Worth a `# COUNTER WRITE: must pass refresh_ttl=True` linter-style comment at the `async_increment_cache` definition, or splitting into two distinct methods (`async_increment_counter` / `async_increment_window`) so misuse is a type error not a missing kwarg. The current API leaves the contract entirely in the caller's head.
- **The `redis_clean_miss = False` then-`True` flag pattern is duplicated** at `proxy_server.py:1797` and `spend_counter_reseed.py:140`. Both spots re-derive the same "Redis returned None → set flag" logic. Extracting a single `redis_get_with_clean_miss(cache, key) -> tuple[Optional[float], bool]` helper would eliminate the duplication and prevent the two sites from drifting.
- **`reset_budget_for_keys_linked_to_budgets` now does an extra `find_many` before the `update_many`** (`:138-153`). That's one DB round-trip per reset cycle — usually fine, but for tenants with thousands of expired-budget-linked keys it could spike DB load on the cron tick. Worth confirming the typical reset batch size and considering a `SELECT ... RETURNING` shaped via Prisma's `update_many` if available, or chunking.
- **The `_invalidate_spend_counter` helper swallows the broad `except Exception as e` at `:74-77`** with a warning log. That's the right policy here (don't fail the budget reset if cache invalidation fails), but the log line `"Failed to reset spend counter %s: %s"` doesn't disambiguate "DB went away while fetching memberships" from "Redis blip during set" — the inner `redis_err` branch above already does that, but the outer catch doesn't. A second log-tag would help operators triage.
- **`refresh_ttl=True` is not applied on `_init_and_increment_spend_counter`'s base-spend warm at `:1976`** wait, it is. Re-checking — yes, `:1975` and `:1980` both pass `refresh_ttl=True`. Good. ✓

## Risk

Medium-low. The TTL-refresh contract is *behavior-changing for the rate-limit window callers* — if any of them silently rely on the legacy bug-shape (e.g. assume "TTL is set on first increment only" for window correctness), making `refresh_ttl=False` the default protects them, but a search across the repo for `async_increment*` callers should confirm nobody else is unintentionally getting the old behavior and depending on it. The two unit tests separately lock both modes which is the right shape; an integration test exercising a multi-pod-style "increment from pod A, miss-reseed-on pod B" scenario would lock Bug 2's fix more durably but the unit-level `redis_clean_miss` flag assertion is acceptable.
