---
pr-url: https://github.com/BerriAI/litellm/pull/26823
sha: d7431c9db953
verdict: merge-after-nits
---

# fix: drop sensitive locals from re-raised error messages

Two-file PII-redaction fix on the error re-raise paths. (1) At `litellm/integrations/prompt_management_base.py:87-93` and `:114-120` (the sync and async twins of `compile_prompt`), the `except Exception as e` re-raise is narrowed from `f"Error compiling prompt: {e}. Prompt id={prompt_id}, prompt_variables={prompt_variables}, client_messages={client_messages}, dynamic_callback_params={dynamic_callback_params}"` down to `f"Error compiling prompt: {e}. Prompt id={prompt_id}"` — i.e. drops `prompt_variables`, `client_messages`, and `dynamic_callback_params` from the rendered ValueError text. (2) At `litellm/litellm_core_utils/llm_response_utils/convert_dict_to_response.py:824-828`, the call-through to `convert_to_model_response_object` (the recursive twin invoked from the error path inside the same function) drops the `hidden_params=hidden_params, _response_headers=_response_headers` keyword arguments before the `raise Exception(...)` re-raise.

Both changes are in the *exception text* path, which is the riskiest place to leak sensitive locals because: error-handling code is the path most likely to be logged at INFO level by downstream operators, exception messages are the part of the response most likely to be surfaced to end-users in proxy mode, the structured-logging path doesn't redact unstructured `str(e)` content, and tracebacks land in Sentry/DataDog with full message contents by default. `prompt_variables` and `client_messages` for a prompt-templating workflow routinely contain end-user PII (the variables *are* the user's data, that's the whole point), so leaking them through error text is a compliance hazard for any operator subject to data-handling regulation.

The fix is the right shape — drop the locals from the rendered text rather than try to redact them, because any redaction strategy (regex on emails/SSNs/etc.) is incomplete by construction for arbitrary-content prompt variables. Two nits that are real:

(1) The PR drops the locals entirely, which removes the *debuggability* that the original verbose message was trying to provide. A halfway shape that keeps debuggability without leaking would be to keep the *types and lengths* (`prompt_variables=<dict with 4 keys>, client_messages=<list of 12 messages>, dynamic_callback_params=<NoneType>`) — operator can correlate "is the failure shape-related or content-related" without seeing a single byte of user data. The current shape forces operators to reproduce locally with the exact `prompt_id` to debug.

(2) `convert_dict_to_response.py:824-828` drops `hidden_params` and `_response_headers` from the *recursive* re-raise call — that's fine for the leak risk, but the absence of a test pinning "the recursive call inside the error path produces the same response shape minus these two fields" means a future "I refactored this to use kwargs unpacking" change can silently re-add them. A 10-line pytest at the call-boundary asserting the recursive call's kwargs is missing.

(3) Stylistic but worth flagging: there's no CHANGELOG note about the error-message text change. Operators who grep their log aggregation for the old verbose text (a common practice for "alert me when prompt compilation fails") will silently lose the alert without realizing it changed.

## what I learned
Exception messages are the most-leaked surface in any LLM-proxy codebase because they sit at the intersection of three correctness anti-patterns: (a) developers default to "include everything that might help debug" when writing error text, (b) operators default to "log at INFO with full message" because the message *is* the structured field for traditional log analysis, (c) error-handling tests are usually shape-tests not content-tests so the leak is invisible to CI. The right fix is almost always "drop the locals, keep the IDs" — debuggability is preserved through correlation IDs, not through inlined values.
