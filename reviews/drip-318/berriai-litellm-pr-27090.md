# Review — BerriAI/litellm#27090 — fix(responses): preserve list-style input during spend-log history reconstruction

- PR: https://github.com/BerriAI/litellm/pull/27090
- Head: `4842059f469efc1060927a91f7a1be4e1b49e210`
- Verdict: **merge-as-is**

## What the change does

One-line fix in
`litellm/responses/litellm_completion_transformation/session_handler.py:119`:
broadens `isinstance(_response_input_param, dict)` to
`isinstance(_response_input_param, (list, dict))`, so that when
`proxy_server_request.input` is a Responses-API list of message blocks
(the shape used by Vertex/Gemini and many SDKs), the system + prior user
turns are replayed instead of silently dropped. Adds a thorough regression
test
(`tests/test_litellm/responses/litellm_completion_transformation/test_session_handler.py::test_get_chat_completion_message_history_with_list_style_input`).

## Strengths

- Test scenario is realistic: `vertex_ai/gemini-2.5-pro`, a system prompt
  with a "secret codename" sentinel, plus a user turn — and asserts both
  roles **and** content survive the round-trip.
- The cast to `ResponseInputParam` is unchanged and still correct, since
  `ResponseInputParam` is a `Union[str, List[ResponseInputItemParam]]`.
- Includes content-extraction logic in the test that tolerates both
  `str` and `list[dict]` content shapes — robust to future shape changes
  in the dataclass.

## Nits (non-blocking)

- Worth a follow-up to also cover the `tuple` case (some serializers
  emit tuples), but `(list, dict)` is the right minimum.
- The fix description correctly identifies the user-visible symptom
  (loss of system persona on `previous_response_id` continuity for
  non-OpenAI providers); consider linking the bug report in the commit
  body for searchability.

Ship it.
