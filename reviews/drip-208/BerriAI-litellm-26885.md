# BerriAI/litellm#26885 — chore: add soniox provider with support for async stt-async-v4 model

- PR: https://github.com/BerriAI/litellm/pull/26885
- Head SHA: `0bc36275f9...`
- Author: dan2k3k4
- Files: ~14 changed (new provider package + tests + registry wiring), ~2100 LOC

## Context

Adds Soniox (https://soniox.com) as a new STT provider, exposing their async transcription pipeline (`stt-async-v4`) through litellm's standard `litellm.transcription()` / `/audio/transcriptions` surface. Soniox's API is multi-step async (file upload → create transcription → poll until status `completed` → fetch transcript → optional cleanup of file + transcription), which doesn't fit `base_llm_http_handler.audio_transcriptions`'s single-request shape. So the PR routes Soniox in `main.py:transcription()` directly to a dedicated `SonioxAudioTranscriptionHandler`, analogous to how OpenAI/Azure transcription handlers are dispatched.

## Design

Four chunks:

1. **New provider package `litellm/llms/soniox/`** — `__init__.py`, `common_utils.py` (env→api-base/api-key resolution, `SonioxException`, defaults `SONIOX_DEFAULT_POLL_INTERVAL=1.0`, `SONIOX_DEFAULT_MAX_POLL_ATTEMPTS=1800` ≈ 30min, `SONIOX_DEFAULT_CLEANUP=["file","transcription"]`), `audio_transcription/handler.py` (687 lines — the orchestrator), `audio_transcription/transformation.py` (request body shaping + response → `TranscriptionResponse`), and a `README.md` documenting the full surface.
2. **Handler structure (`handler.py:320-440`)** — `SonioxAudioTranscriptionHandler.audio_transcriptions` dispatches to sync vs async paths based on `atranscription`. `_prepare()` pulls handler-only kwargs (`soniox_polling_interval`, `soniox_max_polling_attempts`, `soniox_cleanup`, `audio_url`, `file_id`, `filename`) out of `optional_params` (so they aren't forwarded to Soniox) and clamps them with `max(poll_interval, 0.0)` and `max(max_attempts, 1)`. Auth headers come from `provider_config.validate_environment()`.
3. **Registry plumbing** — `litellm/__init__.py` adds `soniox_api_key`, `soniox_models: Set`, scans `model_prices_and_context_window_backup.json` for `litellm_provider == "soniox"` to populate `soniox_models`, joins it into the global model union, registers `"soniox": soniox_models` in the provider→models map. `litellm_core_utils/get_llm_provider_logic.py` adds the `soniox` branch returning `(model, "soniox", api_key, api_base)`. `_lazy_imports_registry.py` registers the transformation module path.
4. **Tests** — three new test modules in `tests/test_litellm/llms/soniox/`: handler integration, transformation unit, provider registration sanity.

## What's good

- **Cleanup runs even on the error path** — the README at line 191 calls this out explicitly, and the handler honors it. This is the right policy for a multi-step API where you'd otherwise leak files in the user's Soniox account if step 3 of 5 throws.
- **Polling clamp** — `max(poll_interval, 0.0)` prevents a misconfigured `0.0` from busy-looping (it'd still poll fast but at least won't be negative); `max(max_attempts, 1)` prevents a zero-attempt no-op. The 1800-attempt × 1s default ≈ 30 min ceiling raises HTTP 504 on overrun rather than hanging forever. Good defensive defaults.
- **`soniox_raw` on `_hidden_params`** — the README at line 203 documents that the full Soniox payload (transcription metadata + raw tokens with timing/speaker/language data) is preserved on `response._hidden_params["soniox_raw"]`, which is the right place — it's outside the OpenAI-compatible `TranscriptionResponse` surface but still accessible to power users.
- **Pricing transparency** — README at line 245 explicitly flags pricing as `0.0` placeholder pending the user's per-second rate from Soniox. Better than silently logging $0 for billing as if it were correct.

## Risks / nits

- **`max_attempts: 1800` default** is generous but means a stuck Soniox job blocks the calling worker for ~30 min before the 504. For LiteLLM proxy users with a 60s gateway timeout in front, the surfaced error will be a gateway timeout, not the 504 with the `transcription_id` for retry. Consider documenting that proxy operators should plumb `soniox_max_polling_attempts` down to match upstream timeout, or default to something tighter (300 = 5 min) and let users opt into longer.
- **`_prepare` mutates `optional_params` in place via `.pop()`** — fine for the current call site, but means a caller who hands the same `optional_params` dict to litellm twice in a row will get different behavior on the second call (the handler-only keys are gone). Document or copy the dict before popping.
- **Substring/`isinstance` cleanup-spec coercion at handler.py:422-428** is correct (handles `None`, `str`, list/tuple/iterable) but doesn't validate the strings against the allowed set `{"file","transcription"}`. A typo like `soniox_cleanup=["files"]` (plural) will silently skip cleanup for that step. Validate against an enum and `raise ValueError` on unknown.
- **No retry on transient 5xx during polling** — the loop just re-issues `GET /v1/transcriptions/{id}` and surfaces the next exception. Soniox's docs probably accept a few transient retries; consider wrapping the poll body in `tenacity` or the existing httpx retry policy.
- **Auth-key env name is `SONIOX_API_KEY`** but the README example at line 184 also references `soniox_api_key=...` as a kwarg. `litellm/__init__.py` adds the module-level `soniox_api_key: Optional[str] = None`. Make sure `validate_environment` precedence is documented: explicit kwarg > module attr > env var (standard litellm precedence), and the test at `test_soniox_provider_registration.py` should assert that order.
- **Missing pass-through route disclaimer in README at line 244** is fine but means users wanting custom Soniox features (real-time WebSocket, language hints, speaker diarization toggles) will hit the wall at v1. Acknowledged as a v1 limitation, which is fine.

## Verdict

**merge-after-nits** — the handler + registry wiring + tests are solid. Need:
1. Validate `soniox_cleanup` strings against the allowed set (silent typo today).
2. Document the `optional_params.pop` mutation contract or copy-before-pop.
3. Either lower the `SONIOX_DEFAULT_MAX_POLL_ATTEMPTS` default to ~300 or add a note to the README that proxy users should match it to upstream gateway timeouts.

The placeholder `0.0` pricing is correct to land that way (it's documented), and the `_hidden_params["soniox_raw"]` plumbing is the right shape.
