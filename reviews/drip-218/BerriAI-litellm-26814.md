---
pr-url: https://github.com/BerriAI/litellm/pull/26814
sha: 55c3129e5f71
verdict: merge-as-is
---

# fix(proxy/batches): forward model to retrieve_batch for bedrock

Fixes a real silent regression on the proxy `/v1/batches/{batch_id}` retrieve path for Bedrock batches: when the proxy decoded `model` from the encoded `batch_id` it did not forward `model=` as a kwarg to `litellm.aretrieve_batch`, so the call fell through `_handle_retrieve_batch_providers_without_provider_config` and 400'd with the ironic error `LiteLLM doesn't support {bedrock} for 'create_batch'` (sic — wrong operation name in the error too). The fix is a 4-line addition at `litellm/proxy/batches_endpoints/endpoints.py:474-478` that sets `data["model"] = model_from_id` immediately after the `data["batch_id"] = data.pop("file_id", original_batch_id)` rename, so the downstream `litellm.aretrieve_batch(**data)` call hits the `BedrockBatchesConfig` provider_config branch instead of the legacy switch.

The companion error-message rewrite at `litellm/batches/main.py:543-555` is also load-bearing — it now correctly says "retrieve_batch" instead of "create_batch", lists the four legacy-switch-supported providers (openai/azure/vertex_ai/anthropic), and explicitly calls out that bedrock requires `model` to be passed. That kind of error-message-as-documentation is the right fix shape because the next person to hit this without `model` will at least know which kwarg is missing.

The 220-line `tests/test_litellm/proxy/test_batch_retrieve_bedrock.py` is exhaustive: it stands up a `Router` with a real bedrock model entry, monkey-patches `litellm.proxy.proxy_server.llm_router`, and round-trips both the retrieve path and the `/v1/files/{file_id}/content` download path with model-encoded IDs (so the regression can't reappear via the file-content side door either).

## what I learned
The right shape for "missing kwarg" regressions is to fix the call site *and* update the error message at the failure point to name the missing kwarg — it costs nothing and saves the next reporter a half-day of bisecting.
