# BerriAI/litellm PR #26758 — Fix/OpenAI min max output tokens

- Repo: BerriAI/litellm
- PR: https://github.com/BerriAI/litellm/pull/26758
- Head SHA: `b75f100f3020ba204787ce1ef2286249c2257e12`
- Author: SipanP (Sipan Petrosyan)
- Size: +7 / −0, 2 files

## Context

The Anthropic-to-Responses translator at
`litellm/llms/anthropic/experimental_pass_through/responses_adapters/transformation.py:333-340`
maps the inbound `max_tokens` to the OpenAI Responses API's
`max_output_tokens`. OpenAI Responses rejects requests where
`max_output_tokens < 16` with a 400. A client that sends an Anthropic-shaped
request with `max_tokens=1` (a real value people use for "just give me a
yes/no token") therefore round-trips into a hard provider error rather
than a one-token completion. The fix clamps the value to `max(max_tokens,
16)` before assignment.

## What the diff does

Two lines of production code:

```python
+ # OpenAI Responses API requires max_output_tokens >= 16
  max_tokens = anthropic_request.get("max_tokens")
  if max_tokens:
+     max_tokens = max(max_tokens, 16)
      responses_kwargs["max_output_tokens"] = max_tokens
```

Plus a focused test at
`tests/test_litellm/.../test_responses_adapters_transformation.py:753-757`:

```python
def test_max_tokens_below_min_clamped_to_16(self):
    req = _make_request(max_tokens=1)
    kwargs = _ADAPTER.translate_request(req)
    assert kwargs["max_output_tokens"] == 16
```

## Concerns

This is a minimal patch and the direction is correct, but the silent-clamp
behavior is the kind of thing that surprises a caller down the line.

1. **Silent clamp vs. observable clamp.** A client that explicitly asked
   for `max_tokens=1` and got back a 16-token completion may interpret the
   extra tokens as the model "ignoring" their cap, or worse, get billed for
   16 tokens of output when their accounting expected 1. At minimum this
   needs a `verbose_logger.debug("clamping max_tokens=%d to OpenAI minimum
   16", max_tokens)` so it's observable in proxy logs. A
   `Litellm-Adjusted-Max-Tokens: original=1,used=16` response header (or
   equivalent in the response metadata) would make it surface to the
   client too.

2. **Constant should be named.** `16` appears as a magic number both in the
   code (`max(max_tokens, 16)`) and in the test (`assert kwargs[...] ==
   16`). A module-level
   `OPENAI_RESPONSES_MIN_MAX_OUTPUT_TOKENS = 16` referenced from both
   sites would make the contract searchable and prevent drift if OpenAI
   ever changes the floor (which they have done before for other limits).

3. **The `if max_tokens:` guard treats `0` as falsy.** A caller sending
   `max_tokens=0` (which OpenAI Responses would also reject for a
   different reason — "must be > 0") will skip the entire branch and
   `max_output_tokens` simply isn't set, falling through to whatever the
   provider default is. That's probably the right thing to do, but it's
   inconsistent with the spirit of the clamp; either clamp 0 → 16 too
   (matches the "ensure valid output tokens" intent) or document that
   `0` is intentionally treated as "unset".

4. **No equivalent fix for the streaming path or other adapters.** The PR
   touches only `transformation.py`. If there's a parallel `streaming` or
   `chat-completions-adapter` path that also forwards `max_tokens` to
   OpenAI, the same 400 will reproduce there. A grep for
   `max_output_tokens` across `litellm/llms/anthropic/` would confirm
   there's only one site.

5. **Test name is fine but the suite is missing the symmetric "no clamp
   when above 16" assertion**. The existing
   `test_max_tokens_mapped_to_max_output_tokens` already pins `512 →
   512`, so the symmetric case is implicitly covered, but a single test
   in the new style (`test_max_tokens_at_boundary_16_unchanged`) makes
   the boundary explicit.

## Suggestions

- Add `verbose_logger.debug` (or a verbose-info log) when the clamp
  fires so it's traceable.
- Promote `16` to a named constant referenced from both code and test.
- Decide explicitly whether `max_tokens=0` should clamp or pass through;
  document the choice.
- Grep for other `max_output_tokens` write sites in the Anthropic
  pass-through to confirm one-PR-fixes-all.

## Verdict

**merge-after-nits** — the clamp is the right behavior and the test pins
the fix. The nits are about making the silent-adjustment observable so
downstream callers don't have to discover it from a token-count mismatch.
