# BerriAI/litellm PR #26459 — [Fix] Reseed enforcement read path from DB on counter miss

- **Repo:** BerriAI/litellm
- **PR:** [#26459](https://github.com/BerriAI/litellm/pull/26459)
- **Head SHA:** `80089a5de329a6202a420291b50a2c5a6a66bc0c`
- **Author:** Michael-RZ-Berri
- **Size:** +101 / −2 across 2 files
- **Reviewer:** Bojun (drip-26)

## Summary

Closes a budget-bypass hole in the multi-pod enforcement path. The
`get_current_spend` helper in `litellm/proxy/proxy_server.py` was
documented as a 3-step fallback chain (Redis → in-memory →
caller's `fallback_spend`), but in practice rung 3 (`fallback_spend`)
is the *in-process* `team_membership.spend` read at request handling
time. In a multi-pod deployment, that value lags behind cross-pod
truth: every Redis TTL expiry hands one request through against a
stale local copy, and the team can intermittently exceed its budget.

The fix interposes a new step 3 between in-memory and
`fallback_spend`: when both counters miss, reseed from the
authoritative DB via `_reseed_spend_from_db(counter_key)`, warm the
counter cache so the next read is fast, and return that DB value.
The original `fallback_spend` becomes step 4 (only used when
prisma is unavailable, i.e. true cold start).

## Key changes

### `litellm/proxy/proxy_server.py:1777–1817` — fallback chain

Doc comment is updated to reflect the new four-step chain:

```
1. Redis counter (cross-pod, authoritative)
2. In-memory counter (single-instance or Redis failure)
3. Reseed from authoritative DB spend (counter expired, cross-pod stale)
4. Caller-supplied fallback (DB unavailable, cold start)
```

The new step 3 (`proxy_server.py:1801–1814`):

```python
db_spend = await _reseed_spend_from_db(counter_key)
if db_spend > 0:
    try:
        await spend_counter_cache.async_increment_cache(
            key=counter_key, value=db_spend
        )
    except Exception:
        verbose_proxy_logger.exception(
            "get_current_spend: failed to warm counter %s after DB reseed",
            counter_key,
        )
    return db_spend

# 4. Final fallback: caller-supplied value (prisma unavailable, cold start)
return fallback_spend
```

A few things worth pinning down on this snippet:

- The cache-warm is best-effort: if `async_increment_cache` raises,
  we log via `exception` and still return `db_spend`. That's
  correct — the read path should not fail because a write-through
  to the counter cache failed.
- `db_spend > 0` is the gate to "use DB" vs "fall through to
  `fallback_spend`". This is the single most important contract in
  the patch and it has a subtle implication discussed in concerns
  below.

### `tests/test_litellm/proxy/test_proxy_server.py:5088–5170` — new tests

Two new tests, both pure unit:

- `test_get_current_spend_reseeds_from_db_when_counter_missing`
  asserts that with both counters empty and DB returning
  `spend=362.0`, `get_current_spend(fallback_spend=30.0)` returns
  `362.0` (i.e. DB wins over the stale in-process fallback) and
  that the counter is warmed with that value.
- `test_get_current_spend_uses_fallback_when_db_unavailable`
  sets `prisma_client = None` and asserts the caller's
  `fallback_spend=15.5` is returned without raising. This guards
  step 4.

The first test is precisely the bypass-mode regression guard; its
docstring spells out the threat model:

> Otherwise, every Redis TTL expiry lets a request through against
> a stale in-process `team_membership.spend`.

That's exactly the bug being fixed and the test pins it down.

## What's good

- Correct framing of the bug. The fix is on the *enforcement read
  path*, not the write path. That's the right surface — write paths
  already round-trip through Redis; the gap was only in how a
  Redis miss was resolved.
- The reseed-and-warm pattern is the right shape: single DB read,
  populate the counter cache so the *next* request hits the fast
  path, and only fall through to the caller's stale value when the
  DB itself is unreachable.
- Best-effort `async_increment_cache` with a logged-exception
  swallow is the right error-handling posture for a write that's
  not on the critical correctness path.
- The two new tests are minimal, hermetic, and assert the exact
  contract that matters (DB beats stale fallback; DB-unavailable
  doesn't raise).

## Concerns

1. **`db_spend > 0` is a load-bearing semantic.** A team or member
   with a *legitimate* spend of exactly `0.0` (brand-new team that
   has just been created but never charged) will skip the DB rung
   and fall through to `fallback_spend`. In the new-team case
   `fallback_spend` will also be `0.0`, so the user-visible
   behaviour is identical — but the *log signal* changes (we'd
   record "DB unavailable, using fallback" when DB was actually
   available and authoritative-zero). Worth either:

   - changing the gate to `if db_spend is not None:` (requires
     `_reseed_spend_from_db` to distinguish "not found" from "found
     and zero"); or
   - adding a comment that the `> 0` gate is intentional and that
     the zero case is observationally indistinguishable from
     fallback.

   Without one of these, future readers will assume the gate is a
   bug and "fix" it, which would change the cache-warming behaviour
   for zero-spend keys.

2. **`async_increment_cache(value=db_spend)` semantics depend on
   the existing key state.** `increment` typically *adds* the value
   to the existing counter rather than *setting* it. If between
   the in-memory miss check at `proxy_server.py:1797` and the
   warm at line 1804 a concurrent request has populated the
   counter, the DB value will be added on top of whatever's already
   there, not replace it — leading to over-counted spend on the
   next request. The PR uses `async_increment_cache` rather than
   `async_set_cache`; need to verify that the underlying
   `DualCache.async_increment_cache` is "set if absent, else
   leave alone" or that the race window is small enough not to
   matter at the proxy's request rate. The test mocks
   `async_increment` and only asserts the call arguments, not the
   resulting cache state, so this race is not covered.

3. **No bound on DB read frequency.** Every Redis miss will now
   issue a DB read. Under a Redis outage where every key misses,
   that's a flood of `litellm_teammembership.find_unique` queries
   exactly when the system is already degraded. Mitigation
   options: a brief in-memory negative cache, or rate-limit the
   `_reseed_spend_from_db` calls per key. Worth at least a
   follow-up ticket.

4. **`_reseed_spend_from_db` is not visible in this diff.** The
   patch adds the *call site* for it, but the function body
   (presumably already in `proxy_server.py` from the earlier
   `test_reseed_spend_from_db_skips_window_variant_keys` test that
   landed in the same file) is the actual workhorse. Reviewers
   need to confirm that function:
   - Distinguishes "key not found" from "spend is zero" (see
     concern #1);
   - Handles `litellm_teamtable` vs `litellm_teammembership` based
     on `counter_key` shape (the existing test name mentions a
     window-variant skip, suggesting parsing is involved);
   - Does not retry indefinitely on prisma failures (defers to
     concern #3).

5. **Test coverage gap: the warm-fail path.** The new test
   asserts the happy-path warm succeeds; it doesn't assert that a
   warm failure still returns `db_spend`. That `try/except` is
   important — a third test that injects a raising
   `async_increment` and asserts `spend == 362.0` would close the
   gap.

## Risk

Medium. The fix is correct in shape, but it introduces a new code
path that fires on the *common* failure mode (Redis miss) under a
*degraded* system state (Redis outage). Concerns #2 and #3 are
operationally significant if either bites.

## Verdict

**merge-after-nits**

The bug is real and the fix is the right shape. Block on:

- A test (or comment) covering the `db_spend == 0` case to nail
  down the `> 0` gate semantic (concern #1).
- A test covering the warm-failure path returning `db_spend`
  (concern #5).
- Maintainer confirmation that `async_increment_cache` is safe
  against concurrent populates here, or a switch to
  `async_set_cache` (concern #2).

The DB-flood concern (#3) can be a follow-up ticket; it's not a
correctness bug, it's a degradation-mode operational concern.

## What I learned

"Cache fallback chain" is one of the highest-value places to write
down a numbered contract in a docstring, because the rungs almost
always have *different correctness properties* (authoritative vs
stale; cross-pod vs single-pod; cold-start vs steady-state). This
PR's update to the doc comment block at lines 1777–1781 is exactly
the kind of in-code contract that prevents the next reader from
"simplifying" rung 3 back into the bypass bug.

The deeper lesson: caller-supplied "fallback" values that come from
*the same in-process state that the cache was supposed to make
authoritative* are not actually fallbacks — they're the bug source
the cache was supposed to mask. When the cache misses, you have to
go to the *real* authority (DB), not back to the in-process
shadow.
