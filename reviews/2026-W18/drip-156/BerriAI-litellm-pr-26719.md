# PR #26719 — fix(passthrough): track spend for interrupted Bedrock streams

- **Repo:** BerriAI/litellm
- **Link:** https://github.com/BerriAI/litellm/pull/26719
- **Author:** mateo-berri
- **State:** OPEN
- **Head SHA:** `b2e2ddc026a0fa83f6ddc0641a3b872649770845`
- **Files:** `litellm/passthrough/main.py` (+ ~50/-15), `litellm/proxy/pass_through_endpoints/streaming_handler.py` (+ ~30/-30), `tests/test_litellm/passthrough/test_streaming_interrupt_spend_tracking.py` (+244/-0)

Resolves LIT-2642 / closes #14457.

## Context

This is a model-class **`GeneratorExit` is not `Exception`** bug. When a Starlette `StreamingResponse`'s consumer disconnects mid-stream (user closes the browser, sends `Ctrl-C` to a CLI client, or closes the TCP connection on a long Bedrock invoke/converse stream), Starlette calls `aclose()` on the response's async generator. CPython then raises **`GeneratorExit`** at the suspended `yield` inside the generator. `GeneratorExit` is a `BaseException`, not an `Exception` — so any `try/except Exception:` wrapping the `async for chunk in ...` loop **does not catch it**.

The previous structure of all three pass-through streaming sites was:

```python
try:
    raw_bytes = []
    async for chunk in response.aiter_bytes():
        raw_bytes.append(chunk)
        yield chunk
    asyncio.create_task(flush(raw_bytes))   # <-- skipped on disconnect
except Exception:
    raise
```

So on every client disconnect, every per-chunk usage byte buffered into `raw_bytes` was silently dropped, and the spend-tracking flush never fired. For Bedrock pass-through, that's measurable money — and for high-throughput Claude-Code-style usage where users routinely interrupt long generations, it's a chronic undercount.

## What changed

Three sites get the same pattern:

### 1. `litellm/passthrough/main.py:_async_streaming` (around line 421-462)

The new structure is:

```python
iter_response = await response

try:
    iter_response.raise_for_status()
except Exception:
    try: await iter_response.aclose()
    except Exception: pass
    raise

raw_bytes: List[bytes] = []
flush_scheduled = False
try:
    async for chunk in iter_response.aiter_bytes():
        raw_bytes.append(chunk)
        yield chunk
except Exception:
    try: await iter_response.aclose()
    except Exception: pass
    raise
finally:
    # GeneratorExit (raised on client disconnect) is not caught by
    # `except Exception`; the finally block ensures partial usage
    # still gets flushed for spend tracking. See LIT-2642.
    if not flush_scheduled and raw_bytes:
        flush_scheduled = True
        try:
            asyncio.create_task(litellm_logging_obj.async_flush_passthrough_collected_chunks(...))
        except Exception as e:
            verbose_logger.exception("Failed to schedule passthrough spend-tracking flush ...")
```

Three structural points worth pulling out:

- **`raise_for_status()` is hoisted out of the chunk-collection block.** Old code raised inside the `try`, which meant a 4xx/5xx upstream error went through the (now `finally`) flush path with `raw_bytes == []`. The hoist preserves the existing test contract (`test_async_streaming_error_propagation.py`: 4xx raises *without* flush) — confirmed by the new test `test_async_streaming_does_not_flush_on_4xx`.
- **`flush_scheduled` flag.** Belt against double-flushing if both the `except Exception:` arm and the `finally:` arm run. Without it, an upstream exception with partial data would schedule the flush task once in the except arm (it doesn't here — the except arm only `aclose`s and re-raises) and again in finally. The flag makes the "exactly once" guarantee structural.
- **`if ... and raw_bytes:` gate.** Don't schedule an empty-payload flush. The new test `test_async_streaming_flushes_on_upstream_exception_with_partial_data` is the cell that verifies "we got 1 chunk before upstream errored ⇒ flush that 1 chunk's worth."

### 2. `litellm/passthrough/main.py:_sync_streaming` (around line 391-414)

Mirror of the async path: hoist `raw_bytes` and `flush_scheduled` outside the `try`, swap the trailing `executor.submit(...)` for a `finally:`-guarded gated submit, swallow scheduling errors with a `verbose_logger.exception` rather than letting them propagate (which would hide the original `GeneratorExit` from the runtime).

### 3. `litellm/proxy/pass_through_endpoints/streaming_handler.py:PassThroughStreamingHandler.chunk_processor` (around line 36-95)

Same pattern at the proxy-side handler. The pre-existing `verbose_proxy_logger.error(f"Error in chunk_processor: {str(e)}")` in the `except Exception` arm is preserved; the post-loop `_route_streaming_logging_to_handler` task scheduling moves into the `finally:` with `logging_scheduled` flag and the `raw_bytes`-non-empty gate. The `end_time = datetime.now()` is recomputed *inside* the finally (rather than just before the task) so it reflects the actual end-of-streaming wall-clock even on disconnect.

### 4. Tests at `tests/test_litellm/passthrough/test_streaming_interrupt_spend_tracking.py` (+244/-0)

Six cells covering the truth table:

- `test_async_streaming_flushes_on_normal_completion` — baseline.
- `test_async_streaming_flushes_on_client_disconnect` — drives the bug via `gen.aclose()` (the same path Starlette takes), asserts only the chunk consumed before disconnect is flushed. This is **the** test for this PR.
- `test_async_streaming_does_not_flush_on_4xx` — pins the existing contract that raised-status doesn't flush.
- `test_async_streaming_flushes_on_upstream_exception_with_partial_data` — the partial-data exception cell.
- `test_sync_streaming_flushes_on_normal_completion` + `test_sync_streaming_flushes_on_early_close` — sync-path mirror.

The `_make_streaming_response` helper at the top of the test file uses `MagicMock(spec=httpx.Response)` with a real `async def _aiter_bytes` generator — so the mock matches the actual `httpx.Response` API surface and the test wouldn't pass against a stale mock if the production type signature changes.

## Design analysis

Three big things this PR gets right:

1. **The fix is at the right layer.** The bug is in *every* streaming pass-through path that buffers chunks for post-flight logging; fixing it at the consumer (e.g. each spend logger having its own client-disconnect detector) would have been an N-place fan-out. Fixing at the buffer-and-flush wrapper is one-place-three-mirrors and structurally correct.

2. **`flush_scheduled` flag plus `raw_bytes` gate is the right invariant pair.** "Exactly once, only if non-empty" is the spend-tracking contract; encoding it as a flag and a gate (rather than `try`-arms) means future code paths (a third except arm, a different finally pattern) can't break the invariant without explicit edits to the flag/gate.

3. **`verbose_logger.exception` on scheduling failure.** If `asyncio.create_task` itself raises (e.g. event loop is shutting down), the original code would let that propagate — which would mask the original `GeneratorExit` and produce confusing stack traces in production logs. The catch here logs the chunk count being dropped and lets the original exit flow through cleanly.

## Risks

1. **`asyncio.create_task` from inside a `finally` during loop shutdown.** On client disconnect, by the time `finally:` runs, the request handler's loop may be in the middle of teardown. `asyncio.create_task` will then raise `RuntimeError: no running event loop` — which is what the wrapping `try/except Exception` catches and logs. So the spend record is still dropped in that worst case; the difference vs the bug is that we get a log line. Worth a code comment confirming this is the intended trade-off vs scheduling synchronously (which would block the disconnect path).

2. **`flush_scheduled` is set *before* the task is created.** If the `asyncio.create_task` raises, the flag is already `True` — but there's nothing afterwards that could re-attempt, so it doesn't matter. Pre-setting the flag is the standard "guard against re-entry within this same finally" idiom and is safe here.

3. **`chunk_processor`'s `verbose_proxy_logger.error("Error in chunk_processor: ...")` log line semantics.** Old code wrapped the entire body in `try/except Exception`; new code wraps only the loop. Errors in `_extract_model_for_cost_injection` (which now runs *outside* the try at lines 41-46) will propagate without that error log. Probably correct (those errors are config errors that should propagate cleanly) but a behavior change worth calling out.

4. **The fix doesn't address the *third* edge case in the canonical `BaseException` hierarchy: `KeyboardInterrupt`.** A `Ctrl-C` to a sync-mode CLI hitting the `_sync_streaming` path will also bypass `except Exception:` — but the new `finally:` catches it too, since `finally:` runs on `BaseException`. Good. Worth a comment in the test file noting that the `gen.aclose()` simulation is the closest reproduction of both `GeneratorExit` and the related `KeyboardInterrupt` family.

## Verdict

**Verdict:** merge-as-is

Model-class fix for a model-class bug. The `BaseException`-vs-`Exception` trap is exactly the kind of subtle Python detail that costs companies real money and is hard to catch without the post-mortem in hand. The three-mirror application across `_async_streaming`/`_sync_streaming`/`chunk_processor` is consistent, the `flush_scheduled` + `raw_bytes` invariant pair is the right encoding, and the test matrix covers the cells that matter (normal completion, disconnect, 4xx, upstream-exception-with-partial-data, sync-mode).

The four nits in *Risks* are documentation/comments, not behavior changes — none block merge.

---

*Reviewed by drip-156.*
