# BerriAI/litellm #26947 — fix metric labels for litellm-side rejects

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/26947
- **HEAD SHA:** `c01dad7622f44140f1b6531ff7bfb88177ad6866`
- **Author:** Michael-RZ-Berri
- **Verdict:** `merge-as-is`

## What the diff does

Two-file Prometheus-label correctness fix for the
`litellm_deployment_failure_responses` metric, splitting the deployment-picked
vs LiteLLM-side-rejected paths so dashboards can distinguish provider 429s
from local limiter rejects (parallel/RPM/TPM/budget).

1. `litellm/integrations/prometheus.py:2009-2031` — introduces a
   `deployment_selected = bool(model_id)` predicate, then conditionally
   populates the `UserAPIKeyLabelValues` enum at `:2018-2031`:
   - **Deployment selected:** keep prior behavior — `litellm_model_name`,
     `model_id`, `api_base`, `api_provider` from the actual deployment, and
     `requested_model = model_group or litellm_model_name`.
   - **LiteLLM-side reject:** all four deployment-scoped labels become
     `""`, and `requested_model = litellm_model_name or model_group or ""`
     so the *requested* model from `request_kwargs["model"]` is captured
     (not the picked-deployment model, which doesn't exist).

2. `litellm/integrations/prometheus.py:2048-2055` — guards the
   `set_deployment_partial_outage(...)` call with the same
   `deployment_selected` predicate. Local-limiter rejects no longer flag
   downstream deployments as in partial outage (the prior behavior would
   have flapped the partial-outage gauge for an empty-string deployment).

3. **Test** `tests/enterprise/litellm_enterprise/enterprise_callbacks/test_prometheus_logging_callbacks.py:734-778` —
   new `test_async_log_failure_event_litellm_side_rate_limit` that:
   - Constructs a `standard_logging_object` with `model_id=""`,
     `model_group=""`, `api_base=""` (the LiteLLM-side-reject shape).
   - Sets `kwargs["model"] = "us/azure/openai/gpt-5-mini"` (the original
     request's `model`).
   - Asserts `set_deployment_partial_outage.assert_not_called()` — pinning
     the partial-outage-gauge-no-flap contract.
   - Asserts the failure-responses labels: `requested_model` carries the
     original `kwargs["model"]`, `litellm_model_name`/`model_id`/
     `api_base`/`api_provider` are all `""`, and `exception_status` is
     `"429"`.

4. **Cosmetic** `:289-296`/`:1564-1572`/`:1666-1674`/`:1750-1758` —
   reformats `with patch(...) as a, patch(...) as b:` chained-context-manager
   blocks into PEP-617 parenthesized `with (...):` form. Pure style.

## Why the change is right

The bug shape is **enum-confusion at the label-emission boundary**: the
prior code emitted *empty* deployment-scoped labels (`litellm_model_name=""`,
`model_id=""`) for LiteLLM-side rejects, but stuffed `model_group or
litellm_model_name` into `requested_model`. For a local rate-limit reject
where no deployment was picked, `model_group` is `""`, `litellm_model_name`
is the requested model — so the emit was technically populated, but the
*deployment-scoped* labels and the *intent-scoped* `requested_model` were
indistinguishable from a real upstream provider 429 (just with empty
strings in some columns). That breaks every operator dashboard that
filters/aggregates on `model_id != ""` to count "real" deployment failures.

The fix's load-bearing detail is the `deployment_selected = bool(model_id)`
predicate at `:2010`. Using `model_id` rather than `model_group` or
`litellm_model_name` is correct because `model_id` is the
deployment-routing-step output (set by the router at deployment-pick
time), while the other two are populated from the input request shape and
exist before any router decision. So `model_id != ""` is the authoritative
"a deployment was actually picked" signal.

The `requested_model` fallback at `:2022` (`label_requested_model =
litellm_model_name or model_group or ""`) is the right capture for the
LiteLLM-side-reject path: `litellm_model_name` here is from
`request_kwargs["model"]` (the user's original ask), which is exactly what
the `requested_model` label is documented to mean.

The partial-outage guard at `:2048-2055` is the silent-but-important fix
— without it, every LiteLLM-side limiter reject would have called
`set_deployment_partial_outage(litellm_model_name="", model_id="",
api_base="", api_provider="")` and flapped the partial-outage gauge under
empty-string keys, polluting the gauge label cardinality.

## Why the test is good

The test covers exactly the load-bearing arms of the fix:
- `set_deployment_partial_outage.assert_not_called()` pins the
  partial-outage-gauge-no-flap contract — without this assertion, the
  partial-outage guard at `:2048-2055` could regress silently.
- The label-by-label assertions at `:773-779` pin the emit-shape contract
  by exact value, so a future "let's just emit `unknown` instead of `""`"
  refactor would surface as a test failure rather than a Prometheus-label
  cardinality explosion in production.

## Verdict rationale

Surgical, well-scoped, well-tested fix at the metric-emission boundary
with the right `deployment_selected` predicate, the right `requested_model`
fallback, and the right partial-outage guard. The test pins both the
positive (correct labels) and negative (no partial-outage flap) contracts.
The cosmetic context-manager reformatting is unrelated noise but doesn't
change semantics.

`merge-as-is`
