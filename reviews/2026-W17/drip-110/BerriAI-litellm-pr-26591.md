# BerriAI/litellm PR #26591 — backfill streaming hidden response cost into metadata

- **PR**: https://github.com/BerriAI/litellm/pull/26591
- **Author**: @milan-berri
- **Head SHA**: `36798c42271374a532a6faac6c7de4382fe514b0`
- **Size**: +58 / −3
- **Files**: `litellm/litellm_core_utils/litellm_logging.py`, `tests/test_litellm/litellm_core_utils/test_litellm_logging.py`

## Summary

Fixes a bug where `metadata.hidden_params.response_cost` was `None` for streaming requests even though the cost had already been computed and stored on the parent `model_call_details["response_cost"]`. The fix: in `_merge_hidden_params_from_response_into_metadata`, before copying `_hidden_params` into the metadata blob, check if `hidden_params["response_cost"]` is `None` and, if so, backfill from `model_call_details["response_cost"]`. Provider-supplied non-`None` costs are preserved.

## Verdict: `merge-as-is`

Small, surgical, and pinned by two complementary tests (the backfill case and the don't-overwrite case). The "preserve provider value" test is the one that distinguishes a careful fix from a sledgehammer; without it a future maintainer would not know that the `if hidden_params.get("response_cost") is None` guard is load-bearing.

## Specific references

- `litellm/litellm_core_utils/litellm_logging.py:1728-1731` — the new three-line backfill:
  ```python
  response_cost = self.model_call_details.get("response_cost")
  if hidden_params.get("response_cost") is None and response_cost is not None:
      hidden_params["response_cost"] = response_cost
  ```
  Correctly ordered: read both first, then guard on both being meaningful before mutating. The guard `is None` (not `not response_cost`) is right — `0.0` is a valid response cost (free-tier embedding, cached hit) and must not be re-backfilled.
- `litellm/litellm_core_utils/litellm_logging.py:1733` — same line previously reassigned `metadata["hidden_params"]` from `getattr(logging_result, "_hidden_params", {})`, throwing away any local mutations. The fix correctly uses the locally bound `hidden_params` (which the new code mutated) instead. This is the actual bug shape: a write into `hidden_params` was being clobbered by a re-read from the response object on the next line.
- `tests/test_litellm/litellm_core_utils/test_litellm_logging.py` — `test_merge_hidden_params_from_response_into_metadata_backfills_response_cost` (model_call_details has `response_cost=0.002`, response has `response_cost=None` → final metadata reads `0.002`) and `test_merge_hidden_params_from_response_into_metadata_preserves_response_cost` (model_call_details `0.002`, response `0.001` → final metadata reads `0.001`, preserving provider's value). Both assertions also check `model_id` is preserved, locking the regression to "response_cost backfill" specifically.

## Nits

None worth blocking on. If anything, a comment one line above the guard explaining *why* (`# Streaming path computes response_cost on model_call_details before _hidden_params is finalized; backfill so dashboards/headers see it.`) would help the next person who tries to "simplify" the fix.

## What I learned

The class of bug here — "I mutated a dict and then immediately reassigned the same key from a stale source" — is exactly the kind that escapes review because both lines look fine in isolation. The fix is also a perfect case study in the value of writing the *negative* test (provider value preserved) alongside the positive one (backfill happens): the positive test alone would let a careless future "simplification" overwrite real provider costs without failing CI. I will steal this two-test pattern for any future "default if missing" fix.
