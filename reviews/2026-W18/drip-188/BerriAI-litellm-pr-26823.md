# BerriAI/litellm#26823 — fix: drop sensitive locals from re-raised error messages

- PR: https://github.com/BerriAI/litellm/pull/26823
- HEAD: `f3fd79b`
- Author: ryan-crabbe-berri
- Files changed: 2 (+2 / -8)

## Summary

Two narrow fixes that stop accidentally surfacing call-site locals into
re-raised exception messages where those locals can carry user prompt
content, prompt-template variables, or response-header dictionaries that
include auth tokens. The first fix shrinks
`prompt_management_base.compile_prompt` / `async_compile_prompt`'s
re-raise to just the prompt id; the second drops `hidden_params` and
`_response_headers` from a kwargs forwarded into the deserializer-error
exception path inside `convert_dict_to_response`.

## Cited hunks

- `litellm/integrations/prompt_management_base.py:90` and `:117` — the
  pre-fix `ValueError` interpolated `prompt_variables`,
  `client_messages`, *and* `dynamic_callback_params` straight into the
  exception message. `client_messages` is the user's chat content;
  `dynamic_callback_params` routinely carries integration credentials
  (Langfuse host/key, custom logger settings). Anywhere this exception
  bubbles to a JSON error response or a structured log sink, that goes
  with it. The replacement `ValueError(f"Error compiling prompt: {e}.
  Prompt id={prompt_id}")` keeps the operator-useful identifier while
  dropping the payload.
- `litellm/litellm_core_utils/llm_response_utils/convert_dict_to_response.py:824-829`
  — pre-fix code passed `hidden_params=hidden_params` and
  `_response_headers=_response_headers` into a downstream call inside
  the `raise Exception(...)` branch. `_response_headers` from upstream
  providers can contain `x-request-id`, `set-cookie`, or in the case of
  Azure-style proxies the `Authorization`/`api-key` header echoed back.
  Removing those two kwargs from the failure path is correct — the
  successful path elsewhere in the file still threads them through to
  the actual `ModelResponse` where they belong.

## Risks

- The first change drops `e` chaining context that may be load-bearing
  for operators debugging template-merge bugs (e.g. "the
  `prompt_template + client_messages` concat blew up because
  prompt_template was None for prompt id X"). Since `{e}` is still
  interpolated in, the *type* of failure remains visible, but the
  *contents* (which message in the list was malformed, what the
  variable substitution looked like) is gone. Worth a `verbose_proxy_logger.debug`
  alongside the `raise` that emits the redacted detail at debug level
  for operators with log access.
- The second change removes parameters from a *raise* path. If the
  callee uses those kwargs to construct a richer exception
  (e.g. `Exception` with attributes the proxy serializer reads later),
  the diff loses that. Quick `git grep` of the called signature will
  confirm — looks safe from the diff context but worth double-checking.
- No regression test in this PR. A unit test that asserts the
  `ValueError` raised from `compile_prompt` does **not** contain a
  known-secret canary string (`"sentinel-secret-value"`) embedded in
  `dynamic_callback_params` would lock the contract against future
  drift back toward "let's add the helpful context back."
- `prompt_id` itself is sometimes a user-supplied string. If the prompt
  id is taken from the request body without validation, an attacker who
  controls it could inject log-line breaking characters or pseudo-fields
  into the error message (`prompt_id=foo\nFAKE_FIELD=...`). Low
  severity, but `prompt_id={prompt_id!r}` would close that.

## Verdict

**merge-after-nits**

## Recommendation

Add a debug-level log restoring the rich context for operator debugging,
quote `{prompt_id!r}` in the format string, and add a unit-test canary;
the underlying redaction is correct and addresses real PII / secret
leakage shapes.
