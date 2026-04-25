# BerriAI/litellm PR #26373 — propagate llm_provider- response headers on Azure Responses API cancel

@3f6cb4af · base `main` · +73/-1 · author `mateo-berri`

## Summary
Fixes #15232. The Azure override `transform_cancel_response_api_response` was building a `ResponsesAPIResponse` directly from JSON without copying provider headers onto `_hidden_params`. As a result the returned object was missing the `llm_provider-`-prefixed headers (e.g. `llm_provider-x-ms-region`) that the rest of the Responses API surface and Azure chat completions both expose for observability/region pinning.

## What changed
- `litellm/llms/azure/responses/transformation.py:348-371` — at top of the cancel transform, late-imports `process_response_headers` then after constructing the response object, sets:
  ```python
  raw_response_headers = dict(raw_response.headers)
  processed_headers = process_response_headers(raw_response_headers)
  response = ResponsesAPIResponse(**raw_response_json)
  response._hidden_params["additional_headers"] = processed_headers
  response._hidden_params["headers"] = raw_response_headers
  return response
  ```
- `tests/test_litellm/llms/azure/response/test_azure_transformation.py:339` — adds `mock_response.headers = {}` to an existing test that previously got away with not having headers.
- `tests/.../test_azure_transformation.py:506-563` — new `test_transform_cancel_response_api_response_propagates_llm_provider_headers` builds a real `httpx.Response` with `x-ms-region`, `x-request-id`, `apim-request-id` and asserts each shows up under the `llm_provider-` prefix.

## Key observations
- Correct fix and the test is exactly the right shape: it uses a real `httpx.Response` (not a plain Mock) so it exercises the real header-merging path, not just the assignment.
- The late `from litellm.litellm_core_utils.core_helpers import process_response_headers` inside the function is fine to avoid a circular import, but worth a one-line comment saying so — otherwise a future cleanup will hoist it to module-level and reintroduce the cycle.
- `_hidden_params["headers"]` stores the *unprocessed* dict, while `additional_headers` stores the prefixed/processed one. That mirrors the convention elsewhere; good consistency.
- Title still says `[WIP]`; body and tests look complete. Either drop the WIP marker or list the remaining items so reviewers know whether to start nitpicking.

## Risks/nits
- The non-cancel paths in the same file should probably get the same treatment if they don't already; quick audit of `transform_*_response_api_response` siblings would confirm. Not blocking — referenced as scope for a follow-up.
- No changelog entry visible. For a behavior add (new headers in response) that's borderline — header *additions* are non-breaking but downstream loggers will start seeing new keys.

**Verdict: merge-after-nits**
