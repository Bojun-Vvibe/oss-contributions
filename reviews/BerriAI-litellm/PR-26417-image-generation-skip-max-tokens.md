# PR #26417 — skip `max_tokens` injection during health check for `image_generation` mode

**Repo:** BerriAI/litellm • **Author:** kimsehwan96 • **Status:** open
• **Net:** +66 / −3

## Change

In `litellm/proxy/health_check.py`,
`_update_litellm_params_for_health_check` now wraps its `max_tokens`
injection in a `model_info.get("mode") != "image_generation"` guard:

```python
if model_info.get("mode", None) != "image_generation":
    _resolved_max_tokens = _resolve_health_check_max_tokens(...)
    if _resolved_max_tokens is not None:
        litellm_params["max_tokens"] = _resolved_max_tokens
```

Adds 4 unit tests covering: (a) image-generation mode skips
`max_tokens`, (b) explicit `health_check_max_tokens` is *also*
suppressed for image mode, (c) chat mode still injects
`max_tokens=5`, (d) no-mode (legacy) still injects.

## What was actually broken

OpenAI's `/v1/images/generations` endpoint validates request body
strictly and rejects unknown fields with `400 "Unknown parameter:
'max_tokens'"`. The health-check helper was unconditionally
injecting `max_tokens=5` for every model, which made
`mode: image_generation` deployments (`dall-e-3`, `gpt-image-1`,
etc.) report **permanently unhealthy** — even though their actual
image-generation calls succeeded. A user looking at `/health` saw
red, paged on-call, and discovered the dashboard was lying.

This is a high-value fix for any production proxy with mixed
chat + image deployments — exactly the litellm sweet spot.

## Design choice worth questioning

The fix lives in `_update_litellm_params_for_health_check`, gated
on `mode == "image_generation"`. There are at least three other
non-chat modes that have the same problem (or might in the future):

- `audio_speech` — TTS endpoints don't accept `max_tokens` either.
- `audio_transcription` — Whisper rejects `max_tokens`.
- `embedding` — OpenAI embeddings reject `max_tokens`.
- `rerank`, `moderation` — same.

The PR notes that `_filter_model_params` "already strips it for
non-chat handlers", which implies the runtime path is already
protected for those modes — so why did `image_generation` slip
through? Worth checking whether this is truly an `image_generation`-
only gap or whether the same gap exists for `audio_speech` and the
fix is incomplete.

If the answer is "the other modes are fine because the audio/embed
handlers strip unknown params silently and OpenAI image is the
only strict one", that's worth a comment in code. Otherwise the
fix should be inverted: only inject `max_tokens` when
`mode in {"chat", "completion", None}`, instead of denylisting
just one mode.

## Why I'd prefer the allowlist

Denylist (`!= "image_generation"`) is correct for today, but every
new strict provider that lands a non-chat mode is a future bug. An
allowlist (`mode in {"chat", "completion", None}` for backward
compat with un-tagged deployments) makes adding a new mode a no-op
instead of a footgun. The cost is exactly one line of additional
churn on the chat side, and the test matrix already covers all
four cases.

## Sharp edge: explicit `health_check_max_tokens` is silently dropped

The second test asserts that **even an explicit operator-set**
`health_check_max_tokens: 50` on an image-mode model is suppressed.
That's the right behavior (the field is invalid downstream), but
operators who set it expecting it to take effect won't see any
warning. The proxy should log a one-time `verbose_logger.info`
when a `health_check_max_tokens` is configured but suppressed by
mode, so debugging "why is my override being ignored" doesn't take
a code dive.

## Verdict

Correct fix for a real production-grade health-dashboard lie.
Test coverage is thorough for the named case. The architectural
question — "should this be a denylist or an allowlist?" — is
worth raising in PR review; the answer doesn't block merge but
the choice will affect every future non-chat handler addition.

## What I learned

Health-check helpers that share code across modes accumulate
mode-specific carve-outs over time. Each carve-out is a denylist
entry. After three or four of them, the inversion to an allowlist
pays for itself — and the cost of the inversion is lowest *now*,
when there's only one carve-out to invert.
