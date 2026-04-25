# BerriAI/litellm PR #26513 — harden /model/info redaction for plural credential field names

@aec7f9ba · base `main` · +397/-8 · author `yuneng-berri`

## Summary
`/model/info` strips secrets via two layers: an explicit `pop(...)` list in `remove_sensitive_info_from_deployment` and a dynamic `SensitiveDataMasker`. The newer `vertex_ai_credentials` field (plural; alongside the long-existing `vertex_credentials`) was on neither list, so deployment configs leaked it verbatim. This PR adds it to the pop list.

## What changed
- `litellm/proxy/.../helpers.py:32` — one-liner add to the explicit redaction list:
  ```python
  deployment_dict["litellm_params"].pop("vertex_ai_credentials", None)
  ```
  alongside the existing `vertex_credentials`, `aws_access_key_id`, `aws_secret_access_key` pops.
- `tests/.../test_helpers.py:88-110` — refactor of `test_remove_sensitive_info_from_deployment_with_excluded_keys` to `copy.deepcopy(base_config)` for each call, fixing test pollution that this PR's reviewer would otherwise hit.
- `tests/test_litellm/test_utils.py:772` — adds `supports_low_reasoning_effort` to the JSON-schema validation list (drift from a sibling PR; flag for split).
- Bulk `model_prices_and_context_window.json` changes — unrelated.

## Key observations
- Correct fix: defense-in-depth here means *both* the dynamic masker and the explicit pop list need to know about plural variants. Adding `vertex_ai_credentials` to the pop list (rather than relying on the masker's pattern) makes the contract explicit and grep-able.
- The deepcopy fix in the test is worth its own micro-PR — without it any test that mutates `model_config` and then re-uses it (which several do further down) is order-dependent. Good catch.
- A more durable fix would be a name-pattern denylist (e.g. anything matching `*_credentials`, `*_key`, `*_secret`) so the next plural/variant naming bug is caught automatically. Right now this is the third or fourth one-off pop addition; the pattern is screaming for generalization.
- The unrelated `model_prices_and_context_window.json` and `supports_low_reasoning_effort` schema additions belong in separate PRs — they make this security fix harder to review and cherry-pick.

## Risks/nits
- Tiny behavior change scoped to `/model/info`. No risk of production blast.
- A regression test that calls `remove_sensitive_info_from_deployment` with `vertex_ai_credentials` populated and asserts it's popped is missing in the visible diff — would harden against a future refactor reverting this.

**Verdict: merge-after-nits**
