---
pr: 26506
repo: BerriAI/litellm
sha: a74772c8ede25a5c110f18c6c8234c0dc5b94264
verdict: merge-as-is
date: 2026-04-25
---

# BerriAI/litellm#26506 — fix(arize): _set_usage_outputs handles raw OpenAI Pydantic CompletionUsage

- **URL**: https://github.com/BerriAI/litellm/pull/26506
- **Author**: alvinttang
- **Closes**: #13672 (252 days old, 18 comments)

## Summary

Fixes a long-standing crash in `litellm/integrations/arize/_utils.py`:
`_set_usage_outputs` calls `usage.get("total_tokens")` and
`usage.get("output_tokens_details", {}).get("reasoning_tokens")`, which
explode with `AttributeError` when `usage` is a raw OpenAI Pydantic
model (`openai.types.completion_usage.CompletionUsage`) — i.e. when the
arize/langfuse_otel logger sees a Chat Completions response that
wasn't normalized to litellm's `Usage`. Same crash on the nested
`completion_tokens_details` / `output_tokens_details` Pydantic objects.

## Reviewable points

- The new helper `_safe_get(obj, key, default=None)` (lines 223–242 of
  the diff) does the right thing in the right order: try `obj.get(key,
  default)` if callable, fall back to `getattr(obj, key, default)`. The
  `try/except TypeError` around the call covers the case where some
  object exposes `.get` with a different signature (rare but real for
  custom adapters). One small concern: a Pydantic v1 model's `.get` is
  technically inherited from `BaseModel.__getattr__` differences; this
  PR's fallback handles that correctly because Pydantic v2 `BaseModel`
  doesn't expose `.get` at all, so it falls through to `getattr`.

- `_set_usage_outputs` itself is rewritten so every formerly-`.get`
  call goes through `_safe_get` (lines 245–263). The chained
  `usage.get("output_tokens_details", {}).get("reasoning_tokens")` is
  refactored into a two-step lookup that explicitly tries
  `completion_tokens_details` (Chat Completions) **then**
  `output_tokens_details` (Responses API). This is a correctness
  upgrade beyond the crash fix — previously the Responses API path
  was the only one looked up, missing reasoning tokens for chat
  completions even when the dict path didn't crash.

- Tests are real, not smoke. `test_set_usage_outputs_pydantic_
  completion_usage` constructs a real
  `openai.types.completion_usage.CompletionUsage` with a
  `CompletionTokensDetails(reasoning_tokens=25)`, asserts up front
  that `not hasattr(usage, "get")` (precondition), and verifies all
  four span attributes (TOTAL=100, PROMPT=40, COMPLETION=60,
  REASONING=25). The second test
  `test_set_usage_outputs_pydantic_response_api_usage` covers the
  Responses API leg with a hand-rolled `PlainResponsesUsage` plus the
  litellm `OutputTokensDetails`. Both tests fail loudly without
  `_safe_get` — exactly the regression contract you want.

- Worth a follow-up but **not blocking**: there's still
  `response_obj and response_obj.get("usage")` at line 240 of the
  changed file that assumes `response_obj` itself is dict-like. If
  arize ever receives a raw OpenAI `ChatCompletion` Pydantic model as
  `response_obj`, that line will repeat the bug one level up. A
  follow-up could route that call through `_safe_get` too.

- The defensive `try/except TypeError` around `getter(key, default)`
  swallows the exception and falls through to `getattr`. In practice
  this is fine, but one could argue logging the unexpected `.get`
  signature at debug level would help diagnose future adapter weirdness.
  Pure nit.

## Rationale

This is a correct, narrow fix for a 252-day-old regression with a
real-world reproduction (langfuse_otel logger). The helper is small,
the two new tests pin down exactly the failure modes that previously
crashed, and the bonus correctness fix (looking at
`completion_tokens_details` for chat completions) closes a quieter
bug. Approve as-is.

## What I learned

Telemetry/observability layers in a wrapper library see the most
heterogeneous object zoo because they observe payloads from every
provider and every internal normalization stage. Coding them
defensively against "is this a dict, a Pydantic v1 model, a Pydantic
v2 model, or a custom adapter?" is not optional — the polymorphism
boundary belongs *inside* the observability layer, not at every
caller.
