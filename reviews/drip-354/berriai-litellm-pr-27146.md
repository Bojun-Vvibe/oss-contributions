# BerriAI/litellm PR #27146 — Litellm client disconnect relay

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/27146
- Head SHA: `9fc9e433f22d8505614c9c60ac249f55c7244ab2`
- Size: +121 / -33 across `litellm/constants.py`,
  `litellm/proxy/common_request_processing.py`,
  `litellm/proxy/proxy_server.py`, plus a new
  `tests/test_litellm/proxy/test_client_disconnection.py`.

## Summary

Replaces the old polling-based
`check_request_disconnection(request, llm_api_call_task)` (which
lived in `proxy_server.py:1773-1801` and is *deleted* in the diff)
with a new variant
`_check_request_disconnection(request, llm_api_call_task,
disconnect_event)` in
`common_request_processing.py:539-571`. The new helper:

1. Reads ASGI messages directly via `await request.receive()`
   (line 555) instead of `await request.is_disconnected()`. This
   sidesteps Starlette's wrapper that sometimes swallows the
   disconnect message during background-task interactions.
2. Sets a shared `asyncio.Event` (`disconnect_event.set()`,
   line 561) so the caller can *distinguish* a client disconnect
   from arbitrary `asyncio.CancelledError` (e.g. from server
   shutdown, downstream timeout, route cancellation).
3. Bounds the lifetime to a configurable
   `DEFAULT_CLIENT_DISCONNECT_CHECK_TIMEOUT_SECONDS` (default 600),
   exposed via env in `constants.py:1311-1314`.

The integration in `base_process_llm_request`
(`common_request_processing.py:1219-1245`) wraps the gathered LLM
call in a try/except that:

- If the gather completes normally, cancels the disconnect task and
  carries on.
- If it raises `CancelledError` and the disconnect event is set,
  re-raises as `HTTPException(status_code=499, "Client disconnected
  the request")`.
- Otherwise re-raises the original `CancelledError`.

`_handle_llm_api_exception` in the same file is updated
(lines 1782-1791) so a `HTTPException(499)` no longer logs a full
stack trace via `verbose_proxy_logger.exception`; it just emits an
`info` line.

## What's good

- The `asyncio.Event` plumbing is the right fix for the original
  bug class: previously, *any* `CancelledError` along the gather
  path (server shutdown, watchdog cancellation, etc.) could not be
  distinguished from a client disconnect, so the 499 path was
  brittle. The new `disconnect_event.is_set()` check at line 1238
  is unambiguous.
- The 499 stack-trace suppression at lines 1782-1791 is a real
  win — `429`/`499`-style "client did this" responses are not
  exceptions in the operator sense and don't belong in `.exception`
  output.
- Default timeout extracted to a constant and env-tunable
  (`constants.py:1311-1314`). Operators with long-running batch
  endpoints can extend the watchdog window without forking.
- Two real `pytest.mark.asyncio` tests in
  `test_client_disconnection.py` cover both happy paths
  (`_with_disconnect`: confirms `cancel()` was called and event was
  set; `_no_disconnect`: confirms neither happened).

## Concerns

1. **`request.receive()` is single-consumer.** ASGI `receive` is a
   *queue*, not a polling primitive. If anything else in the
   request lifecycle (Starlette's body parser, a middleware) is
   *also* awaiting `request.receive()`, this loop will steal the
   `http.request` body chunks and hand them to nobody. Worth
   verifying that for streaming-body endpoints (Vertex / Bedrock
   passthrough, multipart uploads) the body has already been fully
   consumed by FastAPI before `_check_request_disconnection` starts
   awaiting `receive()`. If not, this is a corruption bug, not just
   a noisy warning.
2. **Test for disconnect uses `side_effect = [http.request,
   http.disconnect]`.** That's exactly *one* `http.request` message
   before the disconnect. In real ASGI the broker can deliver many
   `http.request` messages (chunked body) before `http.disconnect`,
   and the loop in lines 553-561 only reads one message per
   `asyncio.sleep(1)` tick. So a fast-disconnecting client behind a
   chunked uploader could take up to ~N seconds to detect, where N
   = number of body chunks. Not a regression vs the old polling
   approach, but worth a comment.
3. **`disconnect_task.cancel()` is called in two places**
   (lines 1232, 1234) but the result is never `awaited`. That's
   normally fine, but mixing it with `try/except CancelledError`
   means the cancel scheduling races with the gather completion.
   Add an `await asyncio.gather(disconnect_task,
   return_exceptions=True)` after cancel to guarantee the task is
   cleaned up before this coroutine returns; otherwise under load
   you'll accumulate orphaned `CancelledError`-pending tasks
   visible in `loop.task_count()`.
4. **`HTTPException` import.** I don't see a new `from fastapi
   import HTTPException` in this diff — the file presumably already
   imports it at module top, but worth a sanity check on
   `common_request_processing.py:1-30` which isn't shown.
5. **Removal of `check_request_disconnection` from
   `proxy_server.py`** (lines 1776-1801 deleted) — anyone importing
   that public-ish symbol from `litellm.proxy.proxy_server`
   directly will break. Add it back as a thin shim that proxies to
   `common_request_processing._check_request_disconnection` (with
   a `DeprecationWarning`) for one minor version, or grep the
   plugin ecosystem before merge.

## Verdict

**merge-after-nits** — the `asyncio.Event` design is correct and
the test coverage is real. Verify the `request.receive()` consumer
collision concern (#1) is non-issue, add the explicit
`disconnect_task` await-on-cancel (#3), and either restore the old
public name as a shim or call out the API change in the changelog
(#5). The 499 log-suppression is a quiet quality-of-life win that
shouldn't get lost in the review noise.
