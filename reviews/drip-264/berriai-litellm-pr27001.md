# BerriAI/litellm PR #27001 — fix: atomic TPM rate limit

- **URL**: https://github.com/anomalyco/litellm/pull/27001
- **Head SHA**: `07b53faf6066`
- **Files touched**: `litellm/proxy/hooks/parallel_request_limiter_v3.py` (substantial — token reservation + atomic check-and-increment refactor)

## Summary

Replaces the read-then-increment TPM enforcement (which races under load) with an upfront token reservation. `_estimate_tokens_for_request` projects `input_tokens + max_tokens` per request; `should_rate_limit` gains `skip_tpm_check=True` for callers using the new reservation path; `atomic_check_and_increment_by_n` regroups keys per-descriptor so each Lua call's keys share a `{hash:tag}` slot (avoiding CROSSSLOT on Redis Cluster). Success/failure callbacks reconcile actual usage against the reservation via stash keys.

## Comments

- `parallel_request_limiter_v3.py:34-38` — `DEFAULT_MAX_TOKENS_ESTIMATE = 4096` and `DEFAULT_CHARS_PER_TOKEN = 4` are baked-in constants. For users running gpt-3.5 with `max_tokens=128k` requests, a 4096 floor will under-reserve by orders of magnitude and TPM will routinely overshoot before reconcile. Make these per-model overrideable via existing model_info.
- `parallel_request_limiter_v3.py:75-89` — the `match`/`case` over `(messages, prompt, input_text)` is clean but Python 3.10+ only. Confirm the project's minimum Python — earlier versions will fail import, not at runtime, which is a deploy-time landmine.
- `parallel_request_limiter_v3.py:99-119` — the contentless-request branch (`max_tokens_estimate = 0`) is correct for the false-429 concern, but a *successful* zero-token reserve still goes through reconciliation. If actual usage comes back >0, the code at the success-callback site (not in this hunk) needs to apply the *full* actual usage to the counter, not `actual - reserved`. The `TPM_RESERVED_SCOPES_KEY` comment (lines 43-49) gestures at this, but only for "scopes without a configured TPM limit" — verify the contentless-but-charged scope case is covered too.
- `parallel_request_limiter_v3.py:160` — `tokens_limit = None if skip_tpm_check else rate_limit.get("tokens_per_unit")` silently disables the TPM check. If a caller sets `skip_tpm_check=True` but forgets to call the new reservation path elsewhere, TPM is *unenforced*. Add a runtime assertion or a structured logger warning when `skip_tpm_check=True` AND no reservation key is observed downstream.
- `parallel_request_limiter_v3.py:185-198` — the "refunds need no atomicity guarantee" claim in the docstring (lines 169-173) is plausible for monotonic refunds, but if a refund and a fresh reservation race, the counter could briefly read negative on Lua peeks. If any consumer surfaces the raw counter value (admin UI, metrics), they'll see noise. Worth a `MAX(0, ...)` clamp in the read path.
- `TPM_RESERVATION_RELEASED_KEY` idempotency marker (lines 50-52): make sure both `async_post_call_failure_hook` and `async_log_failure_event` actually check it before refunding — comment promises the guard but it's not in this hunk.

## Verdict

`request-changes` — the design is correct but several rough edges (per-model max_tokens, Python 3.10+ requirement, silent TPM-disable on `skip_tpm_check`, refund/reservation race observability) need address before this hits production. Critical-path rate-limiter; the bar is high.
