# BerriAI/litellm#26606 — fix(logging): backfill streaming hidden response cost

- PR: https://github.com/BerriAI/litellm/pull/26606
- Head SHA: `50e761d5`
- Diff: +123 / -13 across `litellm_core_utils/litellm_logging.py` (+24/-12), `types/utils.py` (+1/-1), `tests/test_litellm/litellm_core_utils/test_litellm_logging.py` (+98)

## What it does
Two related fixes for streaming responses where the LLM provider returns no `_hidden_params.response_cost`, but litellm has already calculated `model_call_details["response_cost"]`:

1. `litellm_logging.py:1728-1740` — `_merge_hidden_params_from_response_into_metadata` now copies `hidden_params`, backfills `response_cost` from `model_call_details["response_cost"]` if missing, and writes the result into `litellm_params.metadata.hidden_params`. Crucially, it stops mutating the response object's `_hidden_params` (the old code did `getattr(logging_result, "_hidden_params", {})` and assigned a reference).
2. `litellm_logging.py:5480-5493` — same backfill for `clean_hidden_params["response_cost"]` in `get_standard_logging_object_payload`, so the `standard_logging_object.hidden_params.response_cost` that OTEL reads is also populated.
3. `types/utils.py:2662` — widens `StandardLoggingHiddenParams.response_cost` from `Optional[str]` to `Optional[Union[str, float]]`. (The old type was wrong — every callsite uses float.)

The author validated against Jaeger: streaming spans now show `gen_ai.cost.total_cost` and `hidden_params.response_cost` instead of nulls.

## Strengths
- Correct root-cause fix. The previous code overwrote `metadata.hidden_params` with the response's hidden_params reference (`logging_result._hidden_params`), which both leaked across streaming chunks and lost the calculated cost. The new code (a) copies, (b) backfills, (c) preserves provider-supplied non-null values via `if metadata_hidden_params.get("response_cost") is None`.
- Type-widening from `Optional[str]` to `Optional[Union[str, float]]` fixes a long-standing type lie that likely caused mypy noise everywhere this field was touched.
- Four new tests cover (a) backfill happens, (b) standard logging payload backfill, (c) existing non-null is preserved, (d) original response object is not mutated. That last one is the regression test for the silent-mutation bug — important.
- Removed code at `:5443-5447` (old `clean_hidden_params` block) is correctly relocated to `:5482-5493` so the backfill runs before the cost is consumed by `get_model_cost_information`.

## Concerns
- The reordering of `clean_hidden_params` computation from `:5444` to `:5482` (post-`base_model`/`custom_pricing` lines) is functionally fine but moves a non-trivial chunk of logic. A reviewer who only reads the diff in isolation might miss that `raw_response_cost = kwargs.get("response_cost")` (`:5480`) and the `response_cost: float = raw_response_cost or 0.0` line (`:5481`) replace the old `:5499` line. The diff is correct; just call this out in the PR description so it's not obscured.
- The new `metadata = litellm_params.get("metadata") or {}` at `:1738` correctly handles `metadata is None` (the old `setdefault` + `if None` dance was awkward). But if `litellm_params["metadata"]` was previously a different dict object (referenced elsewhere), this swap silently breaks that reference. Probably fine in practice, but worth a quick grep.
- Tests are at `test_litellm_logging.py:2340`+ and look thorough, but they instantiate `LiteLLMLoggingObj` directly. An integration-style test that runs an actual streaming acompletion through a mock provider and asserts the OTEL payload would close the loop. The author already validated manually — capturing that as an automated test would prevent regression.
- `types/utils.py` widening from `Optional[str]` to `Optional[Union[str, float]]` is technically a public type contract change. Downstream consumers using TypedDict for static checks may now see new mypy errors. Almost certainly a non-issue (everyone treats it as a number anyway), but mention in the changelog.

## Verdict
**merge-as-is** — clean fix, real bug, good tests. The notes above are nits and don't block.
