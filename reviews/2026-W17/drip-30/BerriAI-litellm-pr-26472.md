# BerriAI/litellm#26472 — fix(bedrock): avoid duplicate post-call guardrail logs on streaming

- PR: https://github.com/BerriAI/litellm/pull/26472
- Author: shivamrawat1
- +63 / -56
- Base SHA: `70492cee4282541256fb9ac963be94412b1a109c`
- Head SHA: `2b714096c9194ed6349404a56b6e8bab08ede570`
- State: OPEN

## Summary

Threads a new `log_result: bool = True` parameter through
`make_bedrock_api_request` and gates every call to
`add_standard_logging_guardrail_information_to_request_data` on it.
The intent: when the streaming post-call hook fans out to the
guardrail multiple times (one per chunk-batch), only the outer
caller actually logs the guardrail result; inner calls pass
`log_result=False` to suppress duplicates that show up as N
`StandardLoggingGuardrailInformation` rows per request in
LangSmith / DataDog / etc.

## Specific findings

- `litellm/proxy/guardrails/guardrail_hooks/bedrock_guardrails.py:401`
  (head SHA `2b714096`) — new `log_result: bool = True` keyword
  parameter on `make_bedrock_api_request`. Default `True` is the
  right default — keeps every existing direct caller's behavior
  unchanged and forces the streaming caller to opt out.
- `bedrock_guardrails.py:467` — the guardrail-failure error branch
  is wrapped in `if log_result:` *but* the `raise HTTPException(...)`
  still fires unconditionally. Good: the suppression is purely about
  the standard-logging side-channel, not about whether the request
  fails. A request that fails its guardrail will still 4xx
  regardless of `log_result`.
- `bedrock_guardrails.py:487` — the broad `except` branch that
  handles transport-level failures gets the same treatment; the
  `raise` (re-raise of the original exception) stays
  unconditional. Symmetric with the 4xx path — good.
- `bedrock_guardrails.py:502` — the success path's
  `add_standard_logging_guardrail_information_to_request_data` is
  also gated. This is the most important one: the duplication being
  fixed is the success-case "guardrail passed" log emitted once per
  streaming chunk-batch. The `httpx_response = cast(httpx.Response,
  httpx_response)` line right above is unrelated tidying; harmless.
- The diff doesn't show the *caller* that passes `log_result=False`.
  Reviewers should grep the streaming post-call path
  (`async_post_call_streaming_*` or similar) to confirm the
  suppress-callers and ensure exactly one logger fires per logical
  request, not zero.

## Risks

- If the streaming post-call path is the only intended
  `log_result=False` caller but a future refactor moves it, the
  guardrail success will silently stop being logged on streaming.
  A unit test asserting "exactly one
  `StandardLoggingGuardrailInformation` per streaming request"
  would lock this in.
- The error-branch suppression means a streaming request that fails
  its guardrail may not emit *any* error log if every chunk-batch
  hits `log_result=False`. The outer caller must still emit the
  failure log on its own — verify in the missing call-site diff.

## Verdict

`merge-after-nits`

## Rationale

Small, targeted, mechanically correct change. The single missing
piece is the call-site that flips `log_result=False`, which the
diff doesn't include — reviewers must confirm it exists and that
exactly one logger fires per request (not zero on the
error-suppressed streaming path). A regression test would harden
this against future refactors.

## What I learned

"Log exactly once" is the same shape problem as "deliver exactly
once" — both succeed by having an outer coordinator own the side
effect and inner workers stay pure. The `log_result` flag is the
right shape; the test is the missing piece.
