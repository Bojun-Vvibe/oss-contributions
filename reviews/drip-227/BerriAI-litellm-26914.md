---
pr: BerriAI/litellm#26914
title: "Run pre_call_hook on Google generateContent endpoints"
head_sha: 053e0401711e45566d1f1a1388cfe426258eeee5
verdict: merge-after-nits
drip: drip-227
---

## Context

The `/v1beta/models/{model}:generateContent` and
`:streamGenerateContent` proxy endpoints (the Google-native pass-through
shape) bypassed the standard `ProxyBaseLLMRequestProcessing` pipeline.
That meant `pre_call_hook` callbacks — guardrails, PII redaction, prompt
injection guards, custom-logger pre-call handlers — never fired on these
routes, while every other `/v1/chat/completions`-style route had them.
LIT-2706. This PR migrates both endpoints onto the shared base
processor.

## Diff walkthrough

The endpoint refactor at `proxy/google_endpoints/endpoints.py:46-76`
(sync `google_generate_content`) and `:90-138` (`google_stream_*`)
deletes ~85 lines of bespoke setup — `add_litellm_data_to_request`,
manual `function_setup`, manual `litellm_call_id`/`logging_obj` wiring,
manual `StreamingResponse` construction, manual success-headers — and
replaces all of it with one call:

```python
processor = ProxyBaseLLMRequestProcessing(data=data)
return await processor.base_process_llm_request(
    request=request, ...,
    route_type="agenerate_content",  # or "agenerate_content_stream"
    proxy_logging_obj=proxy_logging_obj,
    llm_router=llm_router,
    ...
)
```

That `base_process_llm_request` call already runs `pre_call_hook`
internally (where every other proxy route does), so wiring in is the
fix. The exception arm at `:67-75` and `:130-138` routes failures
through `processor._handle_llm_api_exception` — the same error-mapping
path other endpoints use — instead of the previous bare 500 from
`HTTPException(status_code=500, detail="Router not initialized")`.

The load-bearing field-name translation `generationConfig → config`
(Google's wire format vs the router's kwarg) used to live inline in the
endpoint at `:48-50` of the old file; that block is removed here. The
PR moves the rename into `route_request` (the test at
`test_route_llm_request.py:243-269` proves it now happens there for both
route types). The negative arm at `:272-289`
(`test_route_request_preserves_existing_config_for_google_routes`) pins
the precedence: if the caller already supplied `config`, the
`generationConfig` field must not overwrite it.

## Risks / nits

- The `select_data_generator` symbol is now imported at endpoint scope
  (`endpoints.py:32-39`); a quick grep confirms it's defined in
  `proxy_server`, but the test file doesn't exercise the streaming path
  end-to-end through the new `select_data_generator` plumbing. The
  refactor of `test_google_api_endpoints.py:13-184` deletes ~349 lines
  of the old hand-rolled setup tests; only the `route_request` arms
  remain. Worth a smoke test that an SSE response still comes through
  the new base-processor path with `media_type="text/event-stream"`.
- The exception arm catches `Exception as e` — broad, but matches the
  pattern in sibling endpoints; not a regression.
- Pre-call hook failures will now block these routes the same way they
  block `/v1/chat/completions`. Operators who relied on the previous
  hook-bypass behavior (intentional or not) will see new 4xx/5xx
  responses. This is the intended fix, but worth flagging in release
  notes.
- The body field `route_type` is a plain string; a typo here would
  silently fall through to whatever default `base_process_llm_request`
  has. A `Literal[...]` type would catch it at lint time.

## Verdict

**merge-after-nits** — closes a real callback-coverage gap, deletes
~330 lines of duplicated setup, and the test at `:243-289` pins both
the rename and the precedence contract. Add the SSE smoke test and a
release-note line about the hook now firing on these routes.
