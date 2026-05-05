# BerriAI/litellm #27233 — fix(prometheus): emit real `litellm_remaining_api_key_*_for_model` values when v3 rate limiter is in use

- URL: https://github.com/BerriAI/litellm/pull/27233
- Head SHA: `052f02fa473bee5e32ff1aaa2e629208f140c9a8`
- Diff: +184 / -20 across 2 files (`litellm/integrations/prometheus.py` +59/-20; new `tests/test_litellm/integrations/test_prometheus_remaining_key_model_v3.py` +125/-0)

## Findings

1. Bug is real and accurately diagnosed in the description: `_set_virtual_key_rate_limit_metrics` only reads `metadata["litellm-key-remaining-{requests,tokens}-{model_group}"]`, which the v1 limiter populates. The v3 limiter (`parallel_request_limiter_v3.py`) writes to `response._hidden_params["additional_headers"]` under different keys (`x-ratelimit-model_per_key-remaining-{tokens,requests}`). With v3 enabled, the gauge falls through to `sys.maxsize` (≈9.22e18, displayed as `9e18` in DataDog/Grafana), making the metric useless.
2. Fix at `litellm/integrations/prometheus.py:1408-1445` (in `_set_virtual_key_rate_limit_metrics`): swaps the `or sys.maxsize` short-circuit for explicit `None` checks, then falls back to `kwargs["standard_logging_object"]["hidden_params"]["additional_headers"]` if metadata is empty. The defensive `or {}` chain (`standard_logging_payload.get("hidden_params") or {}`) correctly handles the case where any intermediate key is `None` rather than missing — important because `StandardLoggingPayloadSetup` can emit `hidden_params: None`.
3. Tests at `test_prometheus_remaining_key_model_v3.py` cover the three required cases: v3 path emits real values, v1 metadata path unchanged (regression guard), empty case still falls through. The v1 regression test is the critical one — without it, the swap from `metadata.get(..., sys.maxsize) or sys.maxsize` to two-step `None`-check + assignment could silently change behaviour when metadata holds a valid `0` (the old `or sys.maxsize` would have wrongly clobbered `0`; the new code preserves it). Worth confirming the v1 test exercises the `0` case explicitly.
4. **Drive-by change concern**: the diff also modifies `set_llm_deployment_failure_metrics` (around line 2006-2050 in the diff) to add `deployment_selected = bool(model_id)` branching, routing requested_model into a different label slot when no deployment was picked. This is a separate bug fix (LiteLLM-side rejects had wrong label population) and is not mentioned in the PR description. Should either be split into its own PR or called out explicitly in the description — bundling unrelated fixes makes bisection harder and reviewers may miss it. The change itself looks correct.
5. Minor: the `# noqa: PLR0915` added to `set_llm_deployment_failure_metrics` signals that the function is now too long for pylint's "too-many-statements" rule. Suppressing rather than refactoring is acceptable for a fix PR but should be tracked.

## Verdict

**Verdict:** merge-after-nits
