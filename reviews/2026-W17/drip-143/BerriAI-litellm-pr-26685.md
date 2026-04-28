# BerriAI/litellm#26685 — feat(vector-stores): support Bedrock retrievalConfiguration passthrough

- Head SHA: `6b86e54`
- Author: Sameerlite
- Files: 13 / +156 / −9
- Linear: LIT-2482

## Summary

Threads `extra_body: Optional[Dict[str, Any]]` through the entire vector-store search transformer chain (base + 7 provider transformers: azure_ai, bedrock, gemini, milvus, openai, pg_vector, plus async variants), and adds Bedrock-specific handling at the bedrock transformer that pulls `retrievalConfiguration` (or snake-case `retrieval_configuration`) out of `extra_body`, deep-copies it to avoid mutating the caller's dict, and merges it with `max_num_results`/`filters` from `vector_store_search_optional_params` with verbose-logger warnings on conflict.

## Specific observations

- Base contract change at `litellm/llms/base_llm/vector_store/transformation.py:56-83` — `transform_search_vector_store_request` and `atransform_search_vector_store_request` both grow a required `extra_body: Optional[Dict[str, Any]]` keyword argument. The async helper at `:83-91` correctly forwards `extra_body=extra_body` to the sync override path, preserving the "async-as-sync-with-thread" idiom. All 6 provider implementations updated symmetrically.
- Load-bearing handler-side wiring at `litellm/llms/custom_httpx/llm_http_handler.py:8585`, `:8598`, `:8699` — three call sites (async-async, async-sync, sync) all updated to pass `extra_body=extra_body`. Six-surface symmetry is correct; a missed callsite would have produced a runtime `TypeError: missing required argument`.
- Bedrock-specific merge at `litellm/llms/bedrock/vector_stores/transformation.py:218-258` — the `deepcopy(extra_body.get("retrievalConfiguration") or extra_body.get("retrieval_configuration") or {})` at `:220-225` is the right shape: deep-copy prevents the subsequent `setdefault("vectorSearchConfiguration", {})["numberOfResults"] = max_results` mutation from corrupting the caller's `extra_body` (which would silently survive across requests in long-lived router state).
- Conflict-detection at `:230-236` and `:246-251` — when `vector_store_search_optional_params.max_num_results` (or `filters`) is set AND `extra_body.retrievalConfiguration.vectorSearchConfiguration.numberOfResults` (or `.filter`) is also set with a different value, emit `verbose_logger.debug(...)` naming the override. **Precedence is "optional_params wins over extra_body"** — this is the *opposite* of the usual extra_body-as-escape-hatch semantics in the rest of litellm, where extra_body typically wins for "I know what I'm doing" overrides. Worth a docs callout.
- camelCase + snake_case dual-key acceptance at `:222-225` with `or` fallback is the right ergonomic — Bedrock's native API is camelCase (`retrievalConfiguration`) but Python users will reach for snake_case. The collision case ("user provides both") silently picks camelCase due to short-circuit `or`.
- Type narrowing at `:251-253` — the previous code built `typed_retrieval_config: BedrockKBRetrievalConfiguration = {}` and copied the field explicitly; the new code uses `cast(BedrockKBRetrievalConfiguration, retrieval_config)` directly. This widens the trust boundary — if a future field is added to `BedrockKBRetrievalConfiguration` and the user passes an extra_body field that doesn't match the TypedDict shape, the cast silently passes it through to the API. The previous `typed_retrieval_config["vectorSearchConfiguration"] = ...` selectivity-filter was load-bearing; this refactor removes it. May be intentional ("passthrough means passthrough") but should be named.

## Verdict

`merge-after-nits`

## Rationale

Right shape for the cross-provider plumbing — symmetric extra_body threading through 6 transformers + 3 handler callsites + base contract + async-defaults-to-sync forwarding. Bedrock-specific merge logic correctly deep-copies before mutation and detects conflicts. Nits: (1) document the "optional_params wins over extra_body" precedence — this is opposite to most other extra_body call sites in litellm and will surprise users; (2) the cast-instead-of-explicit-copy at `:251-253` widens the unvalidated-passthrough surface — either restore the explicit field-by-field copy or add a one-line comment naming the trust-the-caller choice; (3) no negative-cell test for "extra_body has retrievalConfiguration with the same `numberOfResults` as max_num_results" — the conflict logger should NOT fire in that case, and the `existing != max_results` guard at `:235` covers it but isn't pinned by a test; (4) PR body has the required Linear ticket but the checklist is unchecked — confirm tests-added line-item before merge. None block.

