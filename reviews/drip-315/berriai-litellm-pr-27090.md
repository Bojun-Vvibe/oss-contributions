# BerriAI/litellm PR #27090 — fix(responses): preserve list-style input during spend-log history reconstruction

- Link: https://github.com/BerriAI/litellm/pull/27090
- Head SHA: `61aa6fbc9e285407697c71426ab3771bea82b257`
- Author: acasademont
- Size: +132 / −1 (2 files)

## Files changed
- `litellm/responses/litellm_completion_transformation/session_handler.py` — `+7 / −1`
- `tests/test_litellm/responses/litellm_completion_transformation/test_session_handler.py` — `+125 / −0` (one new regression test)

## Reasoning

The fix is a one-line widening of an `isinstance` check, with a 125-line regression test that exercises the failure path. This is the right shape for a bug fix.

The actual change at `litellm/responses/litellm_completion_transformation/session_handler.py:118`:

```
elif isinstance(_response_input_param, dict):
    response_input_param = cast(ResponseInputParam, _response_input_param)
```
becomes
```
elif isinstance(_response_input_param, (list, dict)):
    # Standard Responses API shape: `input` is a list of input items...
    response_input_param = cast(ResponseInputParam, _response_input_param)
```

The bug claim — that the standard Responses API ships `input` as a list of input items, and that `ResponsesSessionHandler` was silently dropping list-shaped inputs — is consistent with the OpenAI Responses spec (`EasyInputMessageParam`, `Message`, etc. are explicitly list-typed). Verifiable by reading the upstream spec.

The regression test at `tests/test_litellm/responses/litellm_completion_transformation/test_session_handler.py:391` is well-constructed:
- It seeds a `mock_spend_logs` entry with `input` as a list of `system` + `user` message dicts.
- Asserts `len(messages) == 3` (system + user + assistant) — the original bug would yield 1 message (assistant only).
- Asserts roles `["system", "user", "assistant"]` and that the `ZEPHYR-7` content from the system prompt survives reconstruction.

This is exactly the right kind of test: it captures the regression at the API surface that was broken (`get_chat_completion_message_history_for_previous_response_id`), not at the internal cast.

Minor nits:

1. `litellm/responses/litellm_completion_transformation/session_handler.py:118` widens to `(list, dict)` but the `cast(ResponseInputParam, ...)` is unchanged. If `ResponseInputParam` is a `Union[list[InputItem], dict[...]]`, the cast is sound; if it's `list[InputItem]` only, the previous `dict` branch was already wrong-typed and now both branches share the same incorrect cast. Worth a quick `grep` of `ResponseInputParam` definition to make sure the type union actually accepts both.
2. The new test file's `mock_spend_logs` includes fields that aren't asserted on (e.g. `total_tokens`, `created`, `usage`). Not a blocker, but trimming to the minimum payload would make the test easier to maintain.

## Verdict

`merge-as-is`

One-line fix, real bug, comprehensive regression test, no behavior changes outside the previously-broken branch. The cast nit can be addressed in a follow-up if `ResponseInputParam` doesn't already include `list`.
