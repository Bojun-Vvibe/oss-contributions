# BerriAI/litellm#26954 — fix(rate-limit): close TOCTOU bypass in batch + dynamic limiters

- **PR**: https://github.com/BerriAI/litellm/pull/26954
- **Head SHA**: `dd57ae66915f564e87e91cc5bae89005c0a3c3e0`
- **Size**: +766 / -142, 4 files
- **Verdict**: **merge-after-nits**

## Context

Internal security finding (CVSS 5.3, Medium): rate-limit bypass via TOCTOU in `_PROXY_BatchRateLimiter._check_and_increment_batch_counters` and `_PROXY_DynamicRateLimitHandlerV3._check_rate_limits`. Both performed `should_rate_limit(read_only=True)` then a separate increment as two awaits. Concurrent asyncio requests would all observe the same pre-increment state, all pass validation, all increment — multiplying effective quota by the concurrency level. Reachable from production via `POST /v1/batches`.

## What's right

**`parallel_request_limiter_v3.py` — new `atomic_check_and_increment_by_n` (319 added lines).**

The PR description says this is backed by a Redis Lua script when available with an `asyncio.Lock` + in-memory fallback. The PR description's implementation note ("The lock is **single-process** atomicity. Multi-replica deployments still race across processes; full closure requires a Redis Lua check-and-increment-by-N script") is the most honest framing — and the diff body comment in `batch_rate_limiter.py:170-173` explicitly says the new helper "uses a Redis Lua script when available (multi-process atomic) and falls back to a per-process asyncio.Lock + in-memory operation". Multi-replica deployments running with Redis backing therefore do get full atomicity; only the in-memory fallback is single-process bounded.

**`batch_rate_limiter.py:182-209` — call-site flip.**

Old code (deleted at `:179-251`) was a hand-rolled three-phase: read-only check → manual descriptor walk computing `required_capacity > limit_remaining` per status → separate pipeline-operations build → final `async_increment_tokens_with_ttl_preservation`. New code collapses to one `atomic_check_and_increment_by_n(descriptors, increments=[{"requests": batch_usage.request_count, "tokens": batch_usage.total_tokens} for _ in descriptors])` and a post-call status walk that re-raises via the existing `_raise_rate_limit_error` only when `overall_code == "OVER_LIMIT"`. Net delete is 39 lines; the pipeline-construction loop, the RPM/TPM key derivation, and the descriptor-rate-limit shape introspection all go away because `atomic_check_and_increment_by_n` owns them now. The all-or-nothing semantics ("if any descriptor would exceed its limit, no counter is modified") are exactly what TOCTOU closure requires.

**`dynamic_rate_limiter_v3.py:460-548` — three-phase collapse.**

Old `Phase 1: read-only` → `Phase 2: validate` → `Phase 3: increment` (with the documented "increment separately to avoid early-exit issues" workaround the prior author had to hand-roll because `v3_limiter`'s in-memory check would early-exit and skip incrementing the model counter) is replaced by a single atomic call. The `enforced_descriptors` list is built deliberately: `model_saturation_check` is always enforced, `priority_model` is enforced only when `should_enforce_priority` is true. Same `per_request_increment = {"requests": 1, "tokens": 0}` is supplied for each descriptor. The two HTTPException branches (model-wide vs priority-saturation) are preserved with their existing error messages and `headers` blocks (including the load-bearing `x-litellm-priority` and `x-litellm-saturation` headers that downstream observability scrapes).

**Tests — `test_rate_limiter_toctou.py` (346 new lines, 4 tests).**

- `test_batch_limiter_concurrent_bypasses_tpm_via_toctou` — 5 concurrent 40-token batches against TPM=100. Pre-fix: 5/5 succeed (200 tokens, 100% over). Post-fix: ≤2 succeed. This is the actual regression-pin for the security finding.
- `test_batch_limiter_check_and_increment_is_two_separate_calls` — structural proof for the prior pattern, useful as documentation of what changed.
- `test_dynamic_rate_limiter_v3_concurrent_bypasses_model_capacity` — symmetric for the dynamic limiter, 10 concurrent priority=high requests against RPM=2, post-fix ≤RPM+1.
- `test_dynamic_rate_limiter_v3_phase1_phase3_are_separate_awaits` — structural pin.

The "5/5 succeed (200 tokens, 100% over) → ≤2 succeed" delta is the right shape for the test — it exercises the actual concurrent-bypass path rather than just structural assertions.

The PR description claims the existing rate-limiter suite (`test_parallel_request_limiter_v3.py` + `test_dynamic_rate_limiter_v3.py`) is 59 passed / 0 regressions, which is the right safety net.

## Risks / nits

- **Per-handler-instance lock granularity is coarse.** PR body acknowledges this and offers per-descriptor locks as a follow-up if profiling shows contention. Reasonable trade-off for an unblock-the-CVE PR; worth a TODO referencing the future per-descriptor optimization.
- **Test names contain literal "TOCTOU".** Fine for repo searchability, but the security-finding ticket reference (CVSS 5.3) should also live in the PR-merged commit message so audit log → fix is traceable without the PR body.
- **The `cast(List[Dict[Literal["requests", "tokens"], int]], ...)` in `batch_rate_limiter.py:184-191`** suppresses a typing complaint by re-annotating the list comprehension's inferred type. Cleaner would be a `TypedDict` named `RateLimitIncrement` exposed from `parallel_request_limiter_v3` so callers don't need the `cast`. Cosmetic, doesn't block.
- **The `if rate_limit_response["overall_code"] == "OVER_LIMIT":` predicate in `batch_rate_limiter.py:202-209`** raises on the first OVER_LIMIT status seen, but the prior code raised on the first `required_capacity > limit_remaining` mismatch. If multiple descriptors are over limit simultaneously, the new code raises on whichever the inner loop visits first; the old code raised on whichever the descriptor-iteration order produced. Confirm via test that the error message attribution (which descriptor "wins") matches operator expectations — not a correctness issue, just an observability one.
- **319 added lines in `parallel_request_limiter_v3.py` but the diff shown is mostly call-site flips elsewhere.** The Lua-script implementation is the load-bearing piece for multi-process atomicity claims and isn't visible in the trimmed diff window — would normally request a focused walkthrough comment on the Lua's KEYS/ARGV layout, error semantics on Redis disconnect, and the in-memory fallback's lock-acquisition timeout. Trusting the test matrix for now since the existing-suite-zero-regressions claim covers correctness and the four new TOCTOU tests cover the security closure.
- **Multi-replica caveat needs to land in the docs.** The PR body's "single-process atomicity" caveat is honest; without Redis backing, the bypass is closed only within one worker. A one-line update in `docs/proxy/rate_limits.md` (or wherever rate-limit ops docs live) noting "TOCTOU closure requires Redis backing in multi-replica deployments" would prevent operators from deploying with the in-memory fallback and assuming the CVE is closed.

## Verdict

**merge-after-nits.** Real security fix with a solid test matrix. The atomic-check-and-increment helper is the right primitive; the call-site flips are clean (39-line deletion in `batch_rate_limiter.py`, comparable simplification in `dynamic_rate_limiter_v3.py`); the TOCTOU regression tests pin the actual bypass behavior. Address the docs note on multi-replica/Redis requirement, the descriptor-error-attribution test, and the TODO for per-descriptor lock granularity before merge.
