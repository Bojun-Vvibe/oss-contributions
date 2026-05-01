---
pr: BerriAI/litellm#26949
title: "[Fix] Responses API: Omit Empty Body On DELETE"
head_sha: bd638245e87ce61a6a723080c856cad87e2787d9
verdict: merge-as-is
drip: drip-227
---

## Context

Azure's `DELETE /openai/responses/{id}` endpoint rejects any request
body, including the 2-byte `{}` that httpx serializes when given
`json={}`. The bug: both `delete_response_api_handler` and
`async_delete_response_api_handler` unconditionally pass `json=data`
into `httpx.delete`, where `data` is the empty dict returned by
`transform_delete_response_api_request`. Result: every Azure DELETE
fails with `unexpected_body / Unexpected body with size 2`.

## Diff walkthrough

The fix is symmetric across the async and sync handlers. At
`llm_http_handler.py:2538-2547` (async) and `:2628-2637` (sync) the
single `httpx.delete(url=..., headers=..., json=data, timeout=...)`
call is replaced with a kwargs dict built up conditionally:

```python
delete_kwargs: Dict[str, Any] = {
    "url": url,
    "headers": headers,
    "timeout": timeout,
}
if data:
    delete_kwargs["json"] = data
```

The `if data:` predicate is the load-bearing line — it relies on
Python's truthy semantics, where `{}` is falsy, so an empty dict from
the transformer correctly omits `json=` entirely. Non-empty providers
(if any future Responses-API provider needs a DELETE body) would still
get the body forwarded.

## Tests

`test_llm_http_handler.py:359-424` adds two arms:
`test_async_delete_responses_omits_body_for_azure` and
`test_sync_delete_responses_omits_body_for_azure`. Both patch
`AsyncHTTPHandler.delete` / `HTTPHandler.delete` with a fake that
captures kwargs (`captured.update(kwargs)`), then assert the negative:

```python
assert "json" not in captured
assert "data" not in captured
```

That pair of `not in` assertions is exactly the right shape — it pins
the anti-behavior. A future refactor that "helpfully" sets `json={}`
again would fail the test by name. The URL-shape assertion
`captured["url"].endswith("/openai/responses/resp_xyz?api-version=2025-03-01-preview")`
also incidentally covers the api-version query-string contract.

The `_build_delete_response_mock` helper at `:359-378` correctly
constructs an `httpx.Response` with a synthesized `httpx.Request`, so
the response object is well-formed enough to flow back through the
real handler's response transformer without faking that layer.

## Risks

- The fix is symmetric, but the sync and async branches now have
  identical 8-line copy-paste blocks. Acceptable for a 1-day fix; a
  follow-up could extract a `_build_delete_kwargs` helper.
- `if data:` will also drop a body that happens to be `{}` from a
  non-Azure Responses provider that genuinely wanted to send an empty
  JSON object. No known provider needs that, and OpenAI/Azure both
  reject it, so this is the right default.

## Verdict

**merge-as-is** — minimal, surgical, symmetric, and the negative
assertion pins the contract. Direct fix for a real production-blocking
bug with an Azure error reproduced in the description.
