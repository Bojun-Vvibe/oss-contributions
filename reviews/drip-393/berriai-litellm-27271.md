# Review: BerriAI/litellm#27271

- **PR:** [Fix Prometheus custom metadata label counts (#27268)](https://github.com/BerriAI/litellm/pull/27271)
- **Head SHA:** `2ba2eafcbe42651a1f400a2d15275f8aab1d431e`
- **Merged:** 2026-05-06T03:04:57Z
- **Verdict:** `merge-after-nits`

## Summary

Centralizes "combined custom metadata" assembly via a new `_get_combined_custom_metadata_from_standard_logging_payload()` helper and routes failure logging plus the per-virtual-key rate-limit gauges through `UserAPIKeyLabelValues` + `prometheus_label_factory` so they pick up custom metadata labels consistently. Eliminates "Incorrect label count" errors when `custom_prometheus_metadata_labels` is configured.

## Specific notes

- `litellm/integrations/prometheus.py:1047-1052` — three ad-hoc `metadata.get(...)` lookups + dict merge replaced with a single helper call. Same semantics, less duplication, and now there's one place to fix if metadata source ordering needs to change.
- `litellm/integrations/prometheus.py:1411-1448` — `_set_virtual_key_rate_limit_metrics` rewritten to build `UserAPIKeyLabelValues` and route through `prometheus_label_factory`. Both the requests and tokens gauges now share one `enum_values` object and one `label_context`, so they emit consistent label sets. Previously they used positional `.labels(...)` with hard-coded sanitized values, which silently dropped the custom-metadata columns and caused label-count mismatches when the registry was configured with extras.
- `litellm/integrations/prometheus.py:1569-1591` — `litellm_llm_api_failed_requests_metric` migrated to the same factory pattern via `PrometheusLogger._inc_labeled_counter`. Test in `tests/enterprise/.../test_prometheus_logging_callbacks.py:661-672` updated to assert keyword-style call shape, which is the right contract change.
- `litellm/integrations/prometheus.py:3642-3668` — new helper has good defensive checks (`isinstance(..., dict)` on every nested level). Returns `{}` on non-dict input so callers don't have to special-case `None`. Uses dict-spread merge with the same precedence as the old inline code (`spend_logs` wins over `user_api_key_auth` wins over `requester`). Order preserved — important for not silently changing which value wins for collisions.
- Risk from the Cursor bot summary is real: any dashboard/alert that hard-codes the *old* label tuple will need to either tolerate extra labels or be updated. That's an operator concern, not a correctness concern, but a CHANGELOG note would help.

## Rationale

This is a duplication-removal that also fixes a real bug — exactly the right shape. The new helper is testable in isolation, the call sites are now uniform, and the test was updated rather than deleted. Cardinality risk is the one thing to flag downstream.

Nit: the helper's docstring is a one-liner; expanding it to name the three metadata sources and their precedence would future-proof the file. Not blocking.
