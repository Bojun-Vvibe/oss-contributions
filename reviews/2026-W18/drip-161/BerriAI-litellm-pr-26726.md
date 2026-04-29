# BerriAI/litellm PR #26726 — feat(audio-transcription): add streaming support for whisper + hosted_vllm

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26726
- Head SHA: `55a9d308a2f2eafe370f1367731d47a03c3a1734`
- State: OPEN, +529/-45 across 12 files

## What it does

Threads `stream` through `get_optional_params_transcription` and the OpenAI-whisper / hosted_vllm transformations so audio-transcription endpoints emit SSE chunks (`text/event-stream`) instead of buffering into one `TranscriptionResponse`. Adds:

1. New `transform_audio_transcription_streaming_chunk(chunk: bytes) -> bytes` hook on `BaseAudioTranscriptionConfig` (default pass-through for providers already speaking OpenAI SSE).
2. New `_build_audio_transcription_streaming_response(...)` on `BaseLLMHTTPHandler` that wraps an upstream streaming `httpx.Response` into a `TranscriptionStreamingResponse` with sync + async iterators.
3. `stream` added to `OPENAI_TRANSCRIPTION_PARAMS` (`litellm/constants.py:671`).
4. Whisper / hosted_vllm transformations: filter optional_params to OpenAI-supported set, stringify bools and join lists for httpx multipart compatibility, normalize via `process_audio_file`.
5. Skip the verbose_json override when `stream=True` (verbose_json is incompatible with streaming) but preserve the override otherwise so cost calc still works.

## Specific reads

- `litellm/constants.py:671` — adds `"stream"` to `OPENAI_TRANSCRIPTION_PARAMS`. Correct location; this is the canonical allowlist consumed by `get_optional_params_transcription`.
- `litellm/llms/base_llm/audio_transcription/transformation.py:89-99` — the new abstract hook:
  ```python
  def transform_audio_transcription_streaming_chunk(self, chunk: bytes) -> bytes:
      """Hook for providers to translate streaming chunks to OpenAI-compatible
      SSE bytes. Default: pass-through (provider already speaks OpenAI SSE)."""
      return chunk
  ```
  Right shape — defaults to pass-through, narrow signature (one bytes-in, one bytes-out). Subclass-overrideable per provider. Worth one extra docstring line clarifying that the chunk *is* one full SSE frame (not a sub-frame partial), because providers writing custom translators need to know whether they own re-framing. Currently the contract is implicit in `_aiter`/`_iter`'s `aiter_bytes()` loop, which yields whatever httpx gives back — i.e., *not* guaranteed to be one frame per chunk.
- `litellm/llms/custom_httpx/llm_http_handler.py:1183-1230` — the new `_build_audio_transcription_streaming_response`. Two iterator paths (`is_async`-branched), both `try: yield ... finally: response.aclose()/close()`. Correct lifecycle. The `if not chunk: continue` filter is right (`aiter_bytes()` can yield empty bytes on stream stalls). However the loop calls `transform_audio_transcription_streaming_chunk` on every non-empty chunk including potential keep-alives — providers writing translators that *parse* JSON inside their override will have to defensively handle keep-alive frames.
- The `Iterator` import added at `llm_http_handler.py:10` is used only by `_iter()` — fine.
- Test file `tests/test_litellm/llms/openai/transcriptions/test_audio_transcription_streaming.py` (new in the diff) is the load-bearing artifact. Confirm it covers: (a) async path, (b) sync path, (c) provider override is invoked per chunk, (d) `aclose()`/`close()` runs on both happy-path and exception, (e) `verbose_json` is *not* in the request params when `stream=True`.
- The `verbose_json` skip (referenced in body, not in the diff snippet I read) — verify it's a one-line guard at the optional_params filter, not a deletion of the override (other code paths may rely on `verbose_json` being set when not streaming).

## Risk

- 12 files for a streaming addition is a wide blast radius; concern is parity drift across the four sites the PR touches: `gpt_transformation.py`, `whisper_transformation.py`, `hosted_vllm/transcriptions/transformation.py`, and the proxy server. A future provider added without a streaming override will silently get default pass-through, which fails for non-OpenAI-SSE-compatible providers.
- `proxy_server.py` (changes around line 516) has to wire the `text/event-stream` content-type and the SSE `data: ...\n\n` framing if the upstream doesn't already supply it. The default pass-through assumes upstream framing — confirm the proxy handler doesn't double-frame.
- `TranscriptionStreamingResponse` is a new public type (`litellm/types/utils.py`). Confirm it's exported from `litellm.__all__`/types entry points consistently so SDK consumers get the type for `mypy`.

## Verdict

`merge-after-nits` — solid feature with the right primitive shape (single-method hook on the base config, default pass-through). Three nits: (1) clarify in the hook docstring whether `chunk` is one SSE frame vs an arbitrary byte slice (matters for provider authors), (2) confirm the `verbose_json`-skip is a guarded conditional not a delete, and that the cost-calc path remains intact for non-streaming, (3) verify `TranscriptionStreamingResponse` is exported through the public types surface so SDK users can annotate. Optional: a parametrized test that runs the same scenario through whisper, hosted_vllm, and a third pretend-provider with a custom override, to lock in parity.
