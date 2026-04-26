---
pr: 26551
repo: BerriAI/litellm
title: "fix(guardrails): re-emit chunks in tool_permission streaming hook when no tool_calls found"
url: https://github.com/BerriAI/litellm/pull/26551
head_sha: 9da7cab3b35e6a49c407accaccf4652e9fd3e63a
author: someswar177
verdict: request-changes
date: 2026-04-27
---

# BerriAI/litellm #26551 — re-emit chunks in tool_permission streaming hook when no tool_calls (plus unrelated GCP IAM token-cache change)

Fixes #26547. Two unrelated changes bundled in one PR.

## What the diff actually does

### Change A — the advertised fix

`litellm/proxy/guardrails/guardrail_hooks/tool_permission.py:683-690`:

```python
verbose_proxy_logger.debug(
    "Tool Permission Guardrail: No tool uses found"
)
+mock_response = MockResponseIterator(
+    model_response=assembled_model_response
+)
+async for chunk in mock_response:
+    yield chunk
 return
```

Before: when `assembled_model_response.choices[0].message` had no
`tool_calls`, the async generator hit a bare `return` and yielded
nothing — clients saw `data: [DONE]` only with no message body.

After: the generator wraps the assembled response in
`MockResponseIterator` and yields its chunks before returning, so
plain-text replies pass through.

`tests/test_litellm/proxy/guardrails/guardrail_hooks/test_tool_permission.py:495-541`
— new regression test
`test_async_post_call_streaming_iterator_hook_plain_text_yields_chunks`
that builds a `ModelResponseStream` chunk, an assembled `ModelResponse`
with text-only content, patches `stream_chunk_builder` to return the
assembled response, runs the hook, and asserts
`len(chunks) >= 1`. Comment correctly identifies the bug as "bare
return inside async generator yielded nothing."

### Change B — separate, undocumented

`litellm/_redis_credential_provider.py:1-94` — module-level
`_token_cache: Dict[str, Tuple[str, float]]` keyed by service account,
55-minute TTL via `time.monotonic()`, double-checked locking with
`threading.Lock()`. Refactor turns
`_generate_gcp_iam_access_token` into `_get_cached_gcp_iam_token`
which the sync and async credential paths now call. Reverses the
behavior the *old* test
`test_gcp_iam_credential_provider_regenerates_token_on_each_call`
asserted (each call → fresh token). New tests in
`tests/test_litellm/test_redis.py:213-289`:
`_caches_token`, `_refreshes_on_expiry`,
`_cache_shared_across_instances`, plus an autouse
`clear_gcp_iam_token_cache` fixture.

## Observations

### On change A (the actual subject of the PR)

1. **Fix is correct.** The bare `return` inside an `async def` that
   has `yield` upstream is exactly the bug Python's PEP 525 makes easy
   to write: `return` ends the async iteration immediately, dropping
   any pending content. Routing through `MockResponseIterator` rebuilds
   chunks from the assembled response so the client sees the text.

2. **Test is too weak.** `assert len(chunks) >= 1` doesn't assert the
   *content* of the chunk. If `MockResponseIterator` ever shipped a
   bug that yielded an empty placeholder chunk before the real ones,
   this test would still pass. Tighten to:
   ```python
   assert len(chunks) >= 1
   content = "".join(c.choices[0].delta.content or "" for c in chunks if c.choices)
   assert content == "Hello, world!"
   ```
   Otherwise we're locking "yields anything" not "yields the response."

3. **No coverage for the path where `tool_calls` *does* exist** to
   confirm the fix doesn't regress the rewrite-mode flow. Existing
   `test_async_pre_call_hook_rewrite_mode` is at the pre-call surface,
   not this iterator hook. Worth a sibling streaming test that
   asserts: with tool_calls present, modified chunks are yielded;
   with tool_calls absent, raw assembled chunks are yielded.

4. **`assembled_model_response` is built once and reused.** Confirm
   the path before line 683 actually populates it for the no-tool
   case. From the snippet, looks like `stream_chunk_builder` is
   called above (the test patches it) — but worth glancing at the
   real surrounding code to make sure the no-tool branch isn't
   reached when assembled is `None`. If it can be `None`,
   `MockResponseIterator(model_response=None)` will bomb.

### On change B (the unrelated GCP IAM caching)

5. **This change should be its own PR.** The PR title and body both
   talk only about the tool_permission fix. The Redis credential
   provider change is a substantive behavior change — it intentionally
   *reverses* a previously-tested invariant ("regenerates on each
   call"). Mixing these makes the PR harder to revert if one half
   breaks.

6. **The previous code's contract was load-bearing.** The deleted
   docstring said:
   > generates a fresh GCP IAM token on every new connection. This
   > fixes the 1-hour token expiry issue for async Redis cluster
   > clients, which previously generated the token once at startup
   > and cached it as a static password.
   The new code reintroduces caching, just with a 55-minute TTL.
   That's *probably* fine — Google IAM tokens are valid 60min, so a
   55min cache stays inside the validity window — but it now relies
   on `time.monotonic()` advancing in step with token issuance time
   on the IAM side. If the cache is populated at minute 0 and the
   process is paused (laptop sleep, container suspend) for >5 minutes
   between minute 50 and "first refresh," the cached token's
   `exp` claim has expired but the cache thinks it's valid. The old
   "fresh on every connection" semantics didn't have this hazard.
   Lower TTL to ~50min or check a real expiry from the IAM response
   instead of a heuristic delta.

7. **Module-level cache is shared across all instances *for the same
   process*, which is intended — but also shared across tests** if
   parallel pytest workers reuse the module. The `clear_gcp_iam_token_cache`
   autouse fixture handles that for sequential pytest. For
   pytest-xdist with `loadscope=module`, fine. For other parallel
   modes, the cache is a hidden coupling. Worth a comment.

8. **Double-checked locking is correct here** because reads of a
   `dict` are atomic under the GIL and `_token_cache.get()` returning
   a stale entry just falls through to the locked refresh path. Good.

## Verdict

**request-changes.** Two reasons:

1. **Split the PR.** The Redis IAM caching change is unrelated to the
   tool_permission streaming fix and reverses a previously-tested
   invariant. It deserves its own PR, its own discussion of TTL
   choice (obs 6), and its own merge.
2. **Tighten the test on the actual fix.** `len(chunks) >= 1` is
   too weak; assert content (obs 2), and add a positive-path test
   that confirms the tool-call-present branch still yields modified
   chunks (obs 3).

The tool_permission fix itself is correct and small and would be
merge-as-is on its own with a content-asserting test.

## Banned-string scan

None.
