# BerriAI/litellm #27185 — feat(audio_transcription): add NVIDIA Riva STT provider

- **Head SHA:** `42ced625408c0685b090d5cc76be6632c992a9d1`
- **Base:** `main`
- **Author:** Sameerlite
- **Size:** +1939 / −0 across new `litellm/llms/nvidia_riva/` tree (handler, transformation, audio_utils, common_utils) + integration in `__init__.py`, `main.py`, `get_llm_provider_logic.py`, `types/utils.py`
- **Verdict:** `merge-after-nits`

## Summary

Adds `nvidia_riva` as a new audio transcription provider supporting
both NVCF-hosted (`grpc.nvcf.nvidia.com:443` + `nvcf_function_id`) and
self-hosted Riva ASR via gRPC streaming. Uses
`nvidia-riva-client` SDK at handler-call time so the package isn't a
hard dependency. Adds `LlmProviders.NVIDIA_RIVA` enum entry,
provider-detection wiring, and routing through the existing
`/v1/audio/transcriptions` OpenAI-compatible surface.

## What's right

- **Correct deferred-import pattern.** `transformation.py:81-85`
  explicitly notes "We do *not* construct protobufs here, so this
  module remains importable without `nvidia-riva-client` being
  installed (matching how other providers defer SDK imports to
  handler-call time)" — this is the established convention for
  optional-SDK providers in `litellm/llms/`, and following it means
  `import litellm` doesn't gain a hidden 50-MB dependency for users who
  don't need Riva.

- **OpenAI-API-fidelity handled at the right layer.** The
  `map_openai_params` implementation at `transformation.py:55-81` does
  three correct translations: `language` → Riva's `language_code`
  (with normalization), `timestamp_granularities=["word"]` →
  `enable_word_time_offsets=True` (Riva only natively exposes word
  timing), and a documented note that segment timing is reconstructed
  in the response transformer. This is the right contract — accept
  OpenAI's vocabulary at the boundary, translate to Riva's vocabulary
  internally.

- **Wire-format constants are explicit and documented.**
  `transformation.py:38-41` declares `RIVA_TARGET_SAMPLE_RATE_HZ =
  16000`, `RIVA_TARGET_NUM_CHANNELS = 1`, `RIVA_TARGET_ENCODING =
  "LINEAR_PCM"` as named constants instead of magic numbers buried in
  the audio resampling path. A future contributor adding 8 kHz
  telephony support can grep for `RIVA_TARGET_SAMPLE_RATE_HZ` and find
  every assumption that needs updating.

- **Error class is provider-specific.**
  `NvidiaRivaException` (in `common_utils.py`) extends
  `BaseLLMException`, returned via `get_error_class` at
  `transformation.py:84-89`. This means downstream cost-tracking and
  retry logic that switches on `BaseLLMException` subclass type can
  distinguish Riva failures from generic transport errors.

- **Both deployment shapes covered with one config surface.** The
  config docstring at `transformation.py:43-49` calls out NVCF-hosted
  vs self-hosted explicitly, and the auth path through
  `nvcf_function_id` (NVCF-only) plus `api_base` host:port
  (self-hosted) means a single user config can move between the two
  by changing `api_base` without touching code.

## Nits (worth fixing before merge)

- **gRPC error class doesn't preserve `grpc.StatusCode`.** Riva returns
  rich gRPC status codes (`UNAUTHENTICATED`, `RESOURCE_EXHAUSTED`,
  `INVALID_ARGUMENT`, etc.) that map cleanly onto LiteLLM's standard
  4xx/5xx handling. If `NvidiaRivaException` only carries
  `error_message + status_code + headers` the rate-limit detection in
  `_should_retry` won't fire on `RESOURCE_EXHAUSTED` unless the handler
  manually translates gRPC codes to HTTP equivalents (429 for
  RESOURCE_EXHAUSTED, 401 for UNAUTHENTICATED, 400 for
  INVALID_ARGUMENT, 503 for UNAVAILABLE) before raising. Worth adding a
  small `_GRPC_TO_HTTP` map in `common_utils.py` so retry/fallback
  semantics work the same as for HTTP providers.

- **No graceful failure when `nvidia-riva-client` is missing.** The
  deferred-import pattern is correct, but if a user sets
  `model="nvidia_riva/..."` without installing the SDK they should get
  a clear `ImportError` with a `pip install nvidia-riva-client`
  remediation message at the handler boundary, not a stack trace from
  inside `handler.py:442`. Standard LiteLLM pattern is `except
  ImportError as e: raise BaseLLMException(message="nvidia-riva-client
  is not installed. Run `pip install nvidia-riva-client` to use this
  provider.")`.

- **`response_format` is "stored verbatim" at `transformation.py:69`
  but the supported set isn't documented.** OpenAI accepts `json`,
  `text`, `srt`, `verbose_json`, `vtt` — does Riva's response
  transformer support all five? If not, the user gets a confusing
  empty/wrong response instead of a fail-fast error. Should validate
  against a known set in `map_openai_params` and reject unknowns with
  `drop_params` semantics.

- **`get_supported_openai_params` returns a hardcoded list at
  `transformation.py:51-54`** but doesn't include `prompt`,
  `temperature`, or `model` — fine if Riva genuinely doesn't support
  those, but a one-line code comment explaining the choice (vs OpenAI's
  full param set) would prevent future "why isn't temperature
  respected?" issues.

- **442-line handler is the largest file in the PR — likely contains
  the gRPC streaming loop, audio resampling, and response shaping.**
  Worth confirming it has unit tests or recorded fixture tests for at
  least the format-translation paths (16-bit PCM, MP3→PCM resample,
  multi-channel→mono downmix) since these are the failure modes that
  silently produce garbage transcripts rather than visible errors.

## Risks

- **Hard dep on `nvidia-riva-client` only at runtime — mitigated by
  the deferred-import pattern.** As long as `from
  nvidia_riva_client import ...` lives only inside handler functions,
  not at module top, this PR is safe to merge into a release that
  doesn't ship the SDK in `pyproject.toml`.
- **gRPC channel lifecycle:** if the handler creates a new gRPC
  channel per request, throughput will be terrible (~50ms TLS
  handshake on every call). Worth verifying the handler reuses
  channels per `(api_base, api_key)` tuple.

## Verdict reasoning

`merge-after-nits`: clean architectural fit, correct deferred-import
pattern, well-documented wire-format constants. The gRPC-to-HTTP
status-code mapping and missing-SDK guard are real correctness gaps
for the retry/fallback path but small; the response_format validation
nit is UX. Nothing structural to redo.
