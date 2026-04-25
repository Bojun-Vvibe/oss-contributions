# BerriAI/litellm#26470 — [Fix] Prevent atexit flush hangs and guard proxy_server_request header lookup

- PR: https://github.com/BerriAI/litellm/pull/26470
- Author: yuneng-berri
- +392 / -12
- Base SHA: `70492cee4282541256fb9ac963be94412b1a109c`
- Head SHA: `caec9ed5838538b8ca8276e74cece398e965727e`
- State: OPEN

## Summary

Two unrelated correctness fixes bundled with a model-pricing JSON
update. (1) `get_proxy_server_request_headers` no longer crashes
when `proxy_server_request` is non-dict (e.g. a string left over
from a misbehaving upstream serializer). (2) `_flush_on_exit` in
`logging_worker.py` wraps each per-task coroutine in
`asyncio.wait_for(..., timeout=2.0)` so a single hung custom-callback
coroutine cannot stall Python interpreter shutdown indefinitely.

## Specific findings

- `litellm/litellm_core_utils/llm_request_utils.py:77` (head SHA
  `caec9ed5`) — replaces the chained `.get("proxy_server_request",
  {}).get("headers", {})` with an explicit
  `proxy_server_request = litellm_params.get("proxy_server_request")
  or {}` followed by an `isinstance(..., dict)` check. The previous
  chained-`.get()` would raise `AttributeError` if
  `proxy_server_request` was, say, a JSON string that escaped from
  a serializer. Returning `{}` rather than raising is the right
  defensive default for a header-lookup helper — headers are
  optional everywhere downstream.
- `litellm/litellm_core_utils/logging_worker.py:510` — the
  `_flush_on_exit` loop wraps each `task["coroutine"]` in
  `asyncio.wait_for(..., timeout=2.0)`. Two seconds is generous
  enough that ordinary HTTP loggers will finish but tight enough
  that a hung GCS / S3 / Postgres callback can't keep `atexit`
  spinning forever. The `except Exception: pass` immediately below
  swallows the resulting `asyncio.TimeoutError`, which is the
  correct policy — atexit is best-effort.
- The `model_prices_and_context_window_backup.json` updates are
  large but mechanical: `gpt-5.5` gains
  `*_above_272k_tokens`, `*_flex`, `*_priority`, `*_batches` cost
  variants and `max_input_tokens` jumps from `272000` to
  `1050000`; new `gpt-5.5-2026-04-23` and `gpt-5.5-pro` entries
  appear. These are the kind of pricing entries that *must* match
  upstream — verify against the OpenAI pricing page before merge,
  not after.

## Risks

- Bundling pricing-table updates with two correctness fixes makes
  the PR hard to revert cleanly if one piece regresses. Splitting
  would help, but at this size it's not worth blocking on.
- The 2-second per-task atexit timeout assumes most queued tasks
  complete in milliseconds. If a deployment has e.g. 100 queued
  log tasks and they all timeout, total atexit time is 200s —
  worse than the original "one hung task" symptom for some
  topologies. Consider an outer wall-clock budget across the loop
  in addition to the per-task timeout.

## Verdict

`merge-after-nits`

## Rationale

Both correctness fixes are correct and meaningfully reduce
known production failure modes (interpreter-shutdown hangs and
crashes from non-dict `proxy_server_request`). The pricing JSON
needs a human spot-check against vendor pricing pages. The atexit
loop would benefit from an outer wall-clock budget but the
per-task `wait_for` is already a substantial improvement.

## What I learned

`atexit` callbacks that loop over a queue need *both* per-item and
per-loop time budgets. A per-item timeout alone trades "stuck
forever" for "stuck for N×timeout" when the failure is broad
(network down, etc.).
