# BerriAI/litellm #27003 — fix(health): return 503 when targeted model is unhealthy or DB is disconnected

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27003
- **HEAD SHA:** `3340533cfb5342c11b75e8289db97e8219358c90`
- **Author:** ryan-crabbe-berri
- **Verdict:** `merge-after-nits`

## What the diff does

Two unrelated-but-thematically-grouped HTTP-status fixes in
`litellm/proxy/health_endpoints/_health_endpoints.py`:

### Fix 1: targeted-model `/health?model=foo` should 503 when foo is unhealthy

Previously, if a caller hit `/health?model=foo` against the
background-cached health-check pool and the cache contained a
healthy `bar` and an unhealthy `foo`, the response carried
`healthy_count > 0` (because `bar` was included in the global
aggregate) and the HTTP status stayed 200. Monitoring systems
keying on HTTP status would never page on `foo` being down.

The fix:

1. New helper `_resolve_targeted_model_ids` at `:779-805` —
   resolves a `model` / `model_id` query param to the set of
   deployment IDs to filter by, mirroring the live-path semantics
   in `perform_health_check()` (`model` matches either
   `model_name` alias or `litellm_params.model` provider string;
   `model_id` is taken as-is). Returns `None` for "no targeting,
   no filter."

2. `model_specific_request = bool(model or model_id)` flag at
   `:957`, then in `_post_process` at `:966-971`: if a targeted
   request returned `healthy_count == 0`, set
   `response.status_code = 503` (body shape unchanged).

3. Pre-`_post_process` cache narrowing at `:1014-1062`: when
   targeting is active, intersect the background cache to
   `targeted_ids` *before* `healthy_count` is evaluated — both
   for the scoped-models user-key path (`filter_ids = targeted_ids
   if targeted_ids is not None else allowed_model_ids`) and the
   admin path (new branch at `:1057-1062`).

### Fix 2: `/health/readiness` should 503 when a configured DB is unreachable

In `health_readiness` at `:1442-1497`: the function now takes a
`response: Response` parameter. After the existing
`_db_health_readiness_check()` call, if a DB is configured
(`prisma_client is not None`) AND `db_health_status["status"] !=
"connected"`, set `response.status_code = 503`. The "no DB
configured" path stays 200 — the comment locks the rationale: a
worker without a DB is healthy by design; a worker *with* a
configured DB but no connection cannot serve key/budget/spend
requests and should be taken out of rotation.

Test coverage: existing `test_health_endpoint_*` tests get
`model=None, model_id=None` kwargs (because direct handler calls
bypass FastAPI's `Query()` resolution and would otherwise carry the
truthy sentinel), and a new
`test_health_endpoint_503_for_targeted_unhealthy_model_under_background_cache_admin`
test pins the targeted-503 behavior.

## Why the change is right

Both fixes share the same underlying principle: HTTP status is the
machine-readable signal that monitoring/orchestration systems key
on, and a JSON body that says "healthy_count: 0, this model is
down" while returning 200 forces every consumer to write
body-parsing logic for what is fundamentally a transport-layer
signal.

The `_resolve_targeted_model_ids` helper at `:779-805` is the
right shape — it factors out the model/model_id resolution from
both the cache-narrowing site and the post-process site, sharing
semantics with `perform_health_check()`'s live path so the two
endpoints can't drift.

The pre-`_post_process` narrowing at `:1014-1062` is the load-
bearing detail: if narrowing happened *after* `_post_process`
evaluated `healthy_count`, the targeted-503 path would never fire
because the global aggregate would still see healthy peers.
Putting the filter at the right call-graph position (before the
status evaluation, not after) is what makes the fix work.

The "no DB configured stays 200" carve-out at `:1488-1490` is
correct — a worker that doesn't depend on a DB is fully healthy
without one; only the "configured + unreachable" combination is
the failure signal.

## Nits (non-blocking)

1. **Two unrelated fixes in one PR.** The targeted-503 and the
   readiness-503-on-DB-disconnect are conceptually independent
   (different code paths, different test files, different
   monitoring use cases). They share the "fix the HTTP status to
   match reality" principle but a reviewer interested in only one
   has to pick through both. Consider splitting into #27003a
   (targeted) and #27003b (readiness DB) for cleaner bisecting.

2. **Test mode coverage is incomplete.** The PR adds the targeted-
   503 test for the background-cache + admin path. Missing:
   - non-admin (scoped-key) targeted unhealthy
   - live-path (no `use_background_health_checks`) targeted
     unhealthy (the live path at `:1010+` doesn't go through the
     same code; worth a test that the non-cached path also returns
     503 for targeted unhealthy)
   - readiness-503-on-DB-disconnect (no test added at all that I
     can see — at minimum, mock `_db_health_readiness_check` to
     return `{"status": "Not connected: ..."}` and assert
     `response.status_code == 503` plus that the no-DB-configured
     path still returns 200).

3. **`model_specific_request = bool(model or model_id)` at `:957`** —
   the comment in the test file at `:933-935` notes that direct
   handler calls without going through FastAPI carry a truthy
   `Query()` sentinel for unspecified params. The production code
   at `:957` is fine because real requests go through FastAPI's
   resolution, but a defensive `model is not None and model != ""`
   would be more robust against future direct callers (internal
   tooling, sub-handlers).

4. **`type: ignore[assignment]` additions at `:1478-1479`** are
   unrelated to the documented fix and should either be split out
   or have a one-line "drive-by mypy fix for the index_info
   assignment" line in the PR description so reviewers know they
   weren't snuck in for a separate concern.

5. **`isinstance(targeted_ids, set)` invariant** — the helper
   returns `set` or `None`, but downstream code at `:1051` does
   `targeted_ids if targeted_ids is not None else allowed_model_ids`
   (where `allowed_model_ids` is also a set). Worth a `Set[str]`
   type annotation on the helper signature for documentation.

## Verdict rationale

Two correctness fixes that put HTTP status where it belongs (at
the transport layer, matching body reality), with the load-bearing
pre-`_post_process` narrowing in place and a clean shared helper.
Splitting into two PRs and adding the missing test arms (non-admin
targeted, live-path targeted, readiness-DB-503) are the only real
review items.

`merge-after-nits`
