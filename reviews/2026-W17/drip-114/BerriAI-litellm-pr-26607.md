# BerriAI/litellm PR #26607 — feat(guardrails): add Resemble AI Detect guardrail

- **PR**: https://github.com/BerriAI/litellm/pull/26607
- **HEAD**: `dad97087`
- **Touch**: 6 files, +1896/−~10 (new docs page, new guardrail module, new
  hook init, types entry, dedicated test file)

## Context

New third-party guardrail that scans audio/video/image URLs in LLM
requests against the Resemble Detect deepfake API and blocks the
request when the aggregated score exceeds a threshold or when label
== `fake`. Slot-in for the existing `pre_call` / `during_call` event-hook
shape that other guardrails (Aim, Bedrock GR, etc.) already use.

## Diff

`litellm/proxy/guardrails/guardrail_hooks/resemble/resemble.py:580` is the
implementation. Three things stand out as right:

1. **URL extraction is layered, not first-match.** `_extract_media_urls`
   at `:565-595` pulls from (a) multimodal content parts across *every*
   message — `input_audio.url`, `image_url.url`, Anthropic-style
   `source.url`; (b) regex over joined text in messages + `prompt` +
   `input`; (c) `metadata.mediaUrl` (key configurable via
   `resemble_metadata_key`). The docstring explicitly calls out *why*
   first-match was wrong: "Returning only the first match (the previous
   behaviour) let callers sneak a synthetic URL past the guardrail by
   placing a benign one earlier in the content array." That's the right
   threat-model paragraph to ship in the source.
2. **Fail-open is the default; fail-closed is explicit.**
   `_handle_api_error:543-559` lets the LLM call proceed when Resemble
   itself errors unless `resemble_fail_closed=True` was set. The error
   *is* logged at WARNING. Both choices are pinned in docs.
3. **`MEDIA_URL_REGEX` (`:188`) only matches media-extension HTTPS URLs**
   (`mp3|wav|m4a|...|jpg|jpeg|png|webp|gif`) so the guardrail doesn't
   submit every link in a chat message to Resemble (which would burn
   quota and add latency to every LLM call that mentions a URL).

## Risks / nits

- The `MEDIA_URL_REGEX` is anchored on file *extension*, so a deepfake
  served from `https://cdn.example.com/abc123` (no extension, content
  type set in headers) is invisible to the regex path. The multimodal
  `image_url`/`input_audio` paths catch it, but a freeform text URL
  pointing at an extension-less endpoint slips through. Worth a
  `resemble_strict_extension=False` opt-out that submits any HTTPS URL
  found in text, with the obvious quota-burn warning in docs.
- `poll_timeout_seconds` defaults to 60s and `poll_interval_seconds` to
  2s — that's a 60-second per-request worst-case latency added to every
  LLM call where any media URL is present. For `pre_call` mode this
  blocks the user's response. Recommend the docs call this out
  explicitly and suggest `during_call` mode for latency-sensitive
  setups (the docs already mention `during_call` exists, but don't
  cross-reference the polling timeout).
- The Anthropic-style `source.type == "url"` branch handles `image` and
  `document` part types but not `video` — verify whether Anthropic's
  current API has a video part type and whether it should be included.
- No integration test against a recorded Resemble response; the
  `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_resemble.py`
  file would benefit from a `pytest.fixture(scope="module")` that uses
  `respx` or `httpx_mock` to pin the create-detection-then-poll handshake
  including the "still processing" intermediate state.
- `audio_source_tracing`, `use_reverse_search`, `zero_retention_mode`
  are coerced via `bool(audio_source_tracing)` at `:243-247` — for
  None-as-default these are `False`, which is fine, but the explicit
  `is not None` predicate would be clearer about the three-state nature
  (unset / explicitly off / explicitly on).

## Verdict

Verdict: merge-after-nits
