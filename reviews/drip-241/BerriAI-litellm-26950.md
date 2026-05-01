# BerriAI/litellm#26950 — Fix managed file model_mappings when router resolves a single deployment dict (batch models with id == model_name)

- **PR**: https://github.com/BerriAI/litellm/pull/26950
- **Head SHA**: `b4df0e2c03c14a27714f6f8cd81c259eed63bac9`
- **Files**: `litellm/router_utils/common_utils.py` (+17/-10), `tests/test_litellm/router_utils/test_router_utils_common_utils.py` (+110/-0)
- **Verdict**: **merge-as-is**

## Context

Real correctness bug in `add_model_file_id_mappings` at `litellm/router_utils/common_utils.py:23`. Caller `Router._acreate_file → add_model_file_id_mappings` must produce `Dict[str, str]` (model_id → provider file id) for `LiteLLM_ManagedFileTable`. When `async_get_healthy_deployments` returns a single deployment dict (not a list — happens for batch file uploads where `model_info.id == model_name`), the prior code at `:30-32` treated the dict as if it were already `{model_id: file_id}` and iterated `.items()` — so keys ended up as `litellm_params` / `model_info` with the deployment's *value dicts* as values, not file ID strings. Pydantic then rejected the result with a validation error on `/v1/files`. Reproduced by the bug-screenshot pair in the PR body.

## What's right

- **Root cause correctly identified.** The bug is the type-confusion in the `elif isinstance(healthy_deployments, dict)` branch at the prior `:30-32`: a single-deployment dict is structurally `{"model_name": ..., "litellm_params": ..., "model_info": ...}`, not `{model_id: file_id}`. Iterating `.items()` on it produces `("model_name", "<name>")`, `("litellm_params", {...})`, `("model_info", {...})` — the values are dicts, not strings, hence the Pydantic `LiteLLM_ManagedFileTable` validation failure.
- **Fix normalizes inputs at `:30-39`.** The new code unifies both shapes into a `deployments_list: List[Dict]`, then uniformly extracts `deployment.get("model_info", {}).get("id")` and zips with `responses`. One code path, no shape-dependent branches, structurally cannot regress on a future shape variant because the extraction is always `model_info.id`.
- **`if model_id is not None: model_file_id_mapping[model_id] = response.id` at `:38-39`** correctly skips deployments missing `model_info.id` rather than crashing or producing `{None: file_id}`. Test `test_should_skip_deployment_when_model_info_id_missing` at `:436-456` pins this.
- **Docstring rewritten at `:25-35`** to document the dual-shape input and the rationale: "may be either a list of deployment dicts (multiple matched deployments) or a single deployment dict (when the router resolved a specific deployment, e.g. because the requested model matched a `model_info.id`). Both shapes must be handled by extracting `model_info.id` from each deployment." This is the right discipline — future maintainers see *why* the function accepts both shapes.
- **Test class is structured and named at the contract level.** `TestAddModelFileIdMappings` with five test methods named in the should/when style:
  - `test_should_map_each_deployment_id_when_given_list` (`:381-398`) — happy path.
  - `test_should_extract_model_info_id_when_given_single_deployment_dict` (`:400-417`) — explicit regression docstring at `:401-405` calling out the prior misbehavior.
  - `test_should_handle_batch_model_when_id_matches_model_name` (`:419-449`) — *the* regression case, with explicit docstring "Bug case would have produced keys [`model_name`, `litellm_params`, `model_info`] with non-string values."
  - `test_should_skip_deployment_when_model_info_id_missing` (`:436-456`) — null-id handling.
  - `test_should_return_empty_mapping_when_given_empty_list` (`:458-460`) — empty-input.
- **Test assertions are tight at `:444-446`.** Not just `result == {...}` but also `assert "litellm_params" not in result` and `assert "model_info" not in result` — explicit assertions that the *bug shape* doesn't reappear. This is exactly the right regression-test discipline.
- **`Mock` factory `_make_response(file_id)` at `:368-372`** keeps test setup minimal and readable.

## Risks / nits

- **No test for "list contains a deployment with missing `model_info.id`"** — the skip behavior test only covers the single-deployment-dict case implicitly via the list test at `:436-456`. Worth a one-line test arm but not blocking.
- **`zip(deployments_list, responses)` silently truncates if lengths differ.** If the caller produces N deployments but only M responses (M < N), the function silently drops the extras. Defensive programming would either assert the lengths match or `zip_longest` with explicit handling. Probably out of scope for this fix but worth a follow-up note.
- **Type hint `model_file_id_mapping: Dict[str, str] = {}` at `:36`** is correct but the function signature still says `-> dict` (untyped). Tightening to `-> Dict[str, str]` would let mypy/pyright catch future regressions where a dict-of-dicts sneaks back in.

## Verdict

**merge-as-is.** Tight, well-scoped fix for a real type-confusion bug at the trust boundary between `async_get_healthy_deployments` and `LiteLLM_ManagedFileTable`. The test class structure is exemplary — five specific cases with regression-docstrings calling out the prior bug shape, plus assertions that explicitly check the bug-shape keys are absent. Optional follow-ups (test for list-with-missing-id, length-mismatch handling, tighter return-type annotation) are non-blocking polish.
