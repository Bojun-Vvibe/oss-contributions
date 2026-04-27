# BerriAI/litellm PR #26606 — fix(logging): backfill streaming hidden response cost

- **PR**: https://github.com/BerriAI/litellm/pull/26606
- **HEAD**: `85f4dde2`
- **Touch**: 2 source files + 1 test file, +~50/−~15

## Context

Streaming responses do not carry `_hidden_params["response_cost"]` from
the provider chunks; LiteLLM computes the cost itself and stores it on
`model_call_details["response_cost"]`. But two downstream consumers —
the metadata-merge path (`_merge_hidden_params_from_response_into_metadata`)
and the standard logging payload (`get_standard_logging_object_payload`)
— both read directly from `_hidden_params` for the cost, so for streaming
calls they emitted `response_cost: null` despite a perfectly good cost
sitting on `model_call_details`. Downstream metering/billing dashboards
silently lost streaming cost data.

## Diff

`litellm/litellm_core_utils/litellm_logging.py:1725-1745` — the merge
path now backfills:

```python
metadata_hidden_params = hidden_params.copy()
response_cost = self.model_call_details.get("response_cost")
if (
    metadata_hidden_params.get("response_cost") is None
    and response_cost is not None
):
    metadata_hidden_params["response_cost"] = response_cost
```

Two correct invariants pinned here: (a) `.copy()` so the response's own
`_hidden_params` is not mutated (the `test_merge_hidden_params_..._without_mutating_response`
test asserts `response._hidden_params["response_cost"] is None` after the
merge — exactly the right invariant for a logging path that's supposed
to be read-only on the provider response object), and (b) the
`is None` predicate (not falsy) so `0.0` is preserved, and the
backfill only fills *missing* cost, never overwrites a provider-supplied
one — pinned by `test_merge_hidden_params_..._preserves_response_cost`
which sets `_hidden_params["response_cost"] = 0.001` and
`model_call_details["response_cost"] = 0.002` and asserts the result is
`0.001`.

The same backfill is applied to `get_standard_logging_object_payload`
at `:5479-5489` after the move of `response_cost: float = ...` line up
above the hidden-params build (previously it was *after* and unused for
the build).

The TypedDict change at `litellm/types/utils.py:2662` widens
`response_cost: Optional[str]` to `Optional[Union[str, float]]` — a
correctness fix in its own right since the rest of the codebase
already passes `float`.

## Risks / nits

- The `clean_hidden_params["response_cost"] is None` check at line 5489
  uses dict access not `.get()` — fine because
  `StandardLoggingPayloadSetup.get_hidden_params` is contractually
  expected to return a `StandardLoggingHiddenParams` TypedDict with the
  key always present, but if that contract ever weakens this becomes a
  KeyError at logging time. Defensively `.get("response_cost")` would
  be safer.
- The TypedDict widening from `Optional[str]` to `Optional[Union[str,
  float]]` is the right correction but downstream consumers that
  `json.dumps(hidden_params)` already worked because `float` serializes
  cleanly; any consumer doing `str.startswith()` on the value would
  break. Worth a one-line note in the PR description if any internal
  dashboards do that.
- The three new test functions are well-scoped (positive backfill /
  read-only invariant on response / preserve-when-present) but there's
  no negative test for `model_call_details["response_cost"] is None`
  + `_hidden_params["response_cost"] is None` — should still result
  in `None`, not a KeyError.
- The "move the `response_cost: float = kwargs.get("response_cost", 0)
  or 0.0` line up" change at `:5481-5489` is unrelated to the bug fix
  in shape but necessary to make the backfill use the same value the
  payload-building does — worth a comment.

## Verdict

Verdict: merge-after-nits
