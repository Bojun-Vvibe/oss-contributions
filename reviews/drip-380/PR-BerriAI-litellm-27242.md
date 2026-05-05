# BerriAI/litellm PR #27242 — fix(prometheus): read remaining tokens/requests from additional_headers when v3 limiter populates them

- URL: https://github.com/BerriAI/litellm/pull/27242
- Head SHA: `4c318cd7d7e9acb6f3e774782f239aaa7457b03f`
- Size: +34 / -2

## Summary

Fixes the same `parallel_request_limiter_v3` ↔ Prometheus mismatch covered by drip-379 #27243 / drip-378 #27233 (`litellm_remaining_api_key_{requests,tokens}_for_model` gauges defaulting to `sys.maxsize` ≈ 9.22e18 on v3 because the v3 limiter writes to `hidden_params.additional_headers["x-ratelimit-model_per_key-remaining-{tokens,requests}"]` instead of the legacy `metadata["litellm-key-remaining-{tokens,requests}-{model_group}"]` keys the gauge code reads). This PR is the surgical/minimal variant — no new utility helpers, no refactoring of error metrics, just the gauge fallback path.

## Specific findings

- `litellm/integrations/prometheus.py:1419-1420` — pulls `remaining_requests` and `remaining_tokens` from metadata first (legacy v2 path), preserving zero-cost backwards compatibility.
- `litellm/integrations/prometheus.py:1431-1448` — the load-bearing fallback. When **either** value is `None`, walks `kwargs["standard_logging_object"]["hidden_params"]["additional_headers"]` and reads:
  - `x-ratelimit-model_per_key-remaining-requests`
  - `x-ratelimit-model_per_key-remaining-tokens`
  
  Header names exactly match what `parallel_request_limiter_v3.async_post_call_success_hook` emits (referenced explicitly in the inline comment at `:1426-1428`). 
- `litellm/integrations/prometheus.py:1437-1440` — `try/except AttributeError` around the chained `.get()` calls. Slightly belt-and-braces (the inner `.get(.., {})` and `or {}` already handle `None`/missing keys), but the try guards against `kwargs.get("standard_logging_object")` returning a non-dict-like object — defensible.
- `litellm/integrations/prometheus.py:1450-1456` — final fallback to `sys.maxsize` is `if X is None else X`, replacing the prior `metadata.get(name, sys.maxsize) or sys.maxsize`. Important: the previous `or sys.maxsize` would have **swallowed the legitimate value 0** (a key with zero remaining → reported as maxsize). The new `is None` check correctly preserves `0` as `0`. Quiet bug-fix bundled inside the v3-fallback fix.

## Concerns / nits

- **Overlap with #27243 / #27231 / #27230 / #27229 / #27226**: at least 6 open PRs in the litellm queue address this exact bug (the prior drips already reviewed #27233 + #27243). Maintainers will need to pick one and close the rest. This variant (#27242) is the smallest and most focused — argues for it as the canonical landing.
- Inline comment at `:1421-1430` is excellent; the cross-reference to `parallel_request_limiter_v3.py async_post_call_success_hook` will save the next reader 20 minutes.
- Quietly-fixed `or sys.maxsize` → `is None` semantic improvement deserves either a separate test (assert that a metadata value of `0` is reported as `0`, not maxsize) or a one-line note in the PR body.
- No new tests added. Given the prior PRs in this family have shipped tests for the same fallback path, a maintainer may want to require the test before merge — or accept that tests come in via the canonical PR they pick.

## Verdict

`merge-after-nits` — surgical correctness fix with two real improvements (v3 fallback + `0` not eaten by `or maxsize`). Add a short test pinning the `0`-preservation behavior and the v3-fallback header read, then merge. Maintainer should also reconcile this with the duplicate PRs #27243 / #27231 / #27230 / #27229 / #27226.
