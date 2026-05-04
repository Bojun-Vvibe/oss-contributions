# BerriAI/litellm #27110 — feat(realtime): OpenAI Realtime GA support and beta compatibility

- **Head SHA:** `e33fd0ddcf101d2c8f9ad88ca2f16026988bbb26`
- **Size:** +586 / -69 across 8 files
- **Verdict:** **merge-after-nits**

## Summary
Adds support for the OpenAI Realtime API GA shape on the proxy WebSocket while
keeping legacy beta-shaped clients working. Default upstream connection drops the
`OpenAI-Beta: realtime=v1` header; if the inbound client provides it, the proxy
forwards it upstream. `session.update` flat fields are normalized between GA and
beta, and a GA→beta event/content-type renaming map (`response.output_text.delta`
→ `response.text.delta`, etc.) is applied when the client wants beta.

## Strengths
- `_GA_TO_BETA_EVENT_TYPES` and `_GA_TO_BETA_CONTENT_TYPES` (around line 87 of
  `realtime_streaming.py`) consolidate the renaming surface in two readable maps
  rather than scattered if/elif chains. Easy to extend when OpenAI adds more GA
  events.
- Bidirectional detection (`self._client_wants_beta = self._detect_beta_header(...)`)
  pins the protocol mode at session start, which avoids per-message guessing.
- Adds `conversation.item.added` and `conversation.item.done` to the logged event
  allow-list, matching the new GA naming.
- The catch-all `OpenAIRealtimeStreamResponseBaseObject` fallback for unknown event
  types is the right call — silently dropping new GA events when the schema lags
  upstream is worse than logging them.

## Nits
- The `cast(Dict[str, Any], message)` and `cast(str, message)` calls hide that the
  prior `isinstance` narrow no longer holds inside the `try`. A small explicit
  `assert isinstance(message, str)` (or restructured branches) would be safer than
  silent casts at runtime if a future change breaks the invariant.
- `_AUDIO_FORMAT_MAP` is defined but I do not see it used in the visible diff slice;
  if it is consumed by helpers further down, please cross-reference; if unused,
  drop it.
- Beta-detection is read once at `__init__`. Worth a comment noting that mid-session
  protocol changes are intentionally not supported, so reviewers do not file bugs
  later.
- Test coverage for the GA→beta renaming map is critical; please confirm the new
  tests in this PR exercise at least `response.output_text.delta` →
  `response.text.delta` round-trips both ways.

## Recommendation
Land after confirming `_AUDIO_FORMAT_MAP` is wired up (or remove), tightening the
`cast` calls, and verifying the renaming map has direct test coverage.
