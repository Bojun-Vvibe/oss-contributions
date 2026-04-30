---
pr-url: https://github.com/BerriAI/litellm/pull/26922
sha: b1558ae21917
verdict: merge-as-is
---

# fix(proxy): run model-level post_call guardrails on streaming requests

Six-line surgical fix at `litellm/proxy/utils.py:2275-2280`: inside `async_post_call_streaming_iterator_hook`, before iterating `litellm.callbacks`, the dispatcher now imports `llm_router` from `proxy_server` and calls `_check_and_merge_model_level_guardrails(data=request_data, llm_router=llm_router)` to fold the deployment's `litellm_params.guardrails` into `request_data` so the downstream `should_run_guardrail` gate evaluates the merged set rather than only the user-supplied / key-level / team-level set. The non-streaming path already had this merge; the streaming path was the missed twin — a classic asymmetry bug where two near-identical hooks diverge because only one was touched at feature-introduction time.

The test surface is generous and load-bearing: `tests/test_litellm/proxy/test_model_level_guardrails.py:303-377` is the positive case (model-configured guardrail with `default_on: false` is *not* in the request body but still fires on streaming because the merge runs first), and `:380-426` is the negative case (an unrelated guardrail not configured on the model is *not* fired even after the merge — the gate stays closed for guardrails the model didn't opt into). Both are `@pytest.mark.asyncio` end-to-end through `ProxyLogging.async_post_call_streaming_iterator_hook` with `MagicMock`'d router and a real `CustomGuardrail` subclass that toggles a `was_called` boolean — the right shape for a contract test, no over-mocking of the gate logic itself.

Worth noting the import-inside-function pattern at `:2275` (`from litellm.proxy.proxy_server import llm_router`) — that's the established pattern in this file to dodge the proxy_server ↔ utils circular import, so it's correct here, but it does mean a missing `llm_router` (proxy not initialised) raises `ImportError` mid-stream rather than degrading. Acceptable because every code path that reaches `async_post_call_streaming_iterator_hook` has already gone through proxy init.

## what I learned
When a request-mutation step exists on the non-streaming hook, the streaming-iterator hook needs the *exact same* mutation, in the *exact same place*, and ideally factored into a shared helper so the next reviewer doesn't have to spot the divergence by eye. The two near-identical mock-router test pairs in this PR are the right way to lock the symmetry in CI.
