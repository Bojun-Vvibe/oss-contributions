# BerriAI/litellm #27142 — feat: use W3C traceparent header trace-id as session_id for call chaining

- **Repo**: BerriAI/litellm
- **PR**: [#27142](https://github.com/BerriAI/litellm/pull/27142)
- **Author**: krrish-berri-2
- **Head SHA**: `5e52132381570b148904fab0a86d7779307ca09b`
- **Base branch**: `litellm_internal_staging`
- **Created**: 2026-05-05T00:06:35Z

## Verdict

`request-changes`

## Summary of change

Adds W3C `traceparent` as a fourth fallback source for `session_id` /
`trace_id` in `litellm/proxy/litellm_pre_call_utils.py`:

- `litellm_pre_call_utils.py:320-322` (docstring) — declares the new behavior
  as priority 4, after `x-litellm-trace-id`, `x-litellm-session-id`, and
  generic `x-<vendor>-session-id` headers.
- `litellm_pre_call_utils.py:338-339` — the `or normalized.get("traceparent")
  or None` is appended to the existing chain in `get_chain_id_from_headers`.

Also adds 6 unit tests in `tests/test_litellm/proxy/test_litellm_pre_call_utils.py:2279-2345`:
case-insensitive matching, empty-string ignored, lower priority than explicit
litellm headers and lower priority than generic vendor session headers, plus
a metadata-population integration test that asserts both `session_id` and
`trace_id` get set on the request data.

## What's good

- The intent — letting agent stacks that already emit W3C tracecontext chain
  their LLM calls without an extra header — is clearly the right direction
  and aligns with OpenTelemetry conventions.
- Priority ordering is sensible: explicit litellm headers > vendor session id
  > traceparent. The two precedence tests on lines 2304 and 2317 lock that in.
- Empty-string is ignored (line 2295 test). Good.
- The integration test on `:2331-2345` proves both `session_id` and `trace_id`
  end up in metadata with the same value.

## Significant concerns

1. **Whole `traceparent` value used as session_id is the wrong granularity.**
   A traceparent looks like
   `00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01`. The four fields
   are `version-traceid-spanid-flags`. The intent (per the PR description: "use
   the trace-id for call chaining") is to chain calls that share a *trace*,
   but this PR uses the whole header — including the per-call *span id* and
   *flags* — as the session id. Two calls in the same trace will have
   different span ids and therefore different session ids, which defeats the
   stated purpose. The test on line 2284 even names the variable
   `traceparent` and uses the entire string as the expected session id, which
   means the test is locking in the bug. **The fix is to parse out the
   trace-id (`parts[1]`) only**, after a basic format check (4 dash-separated
   fields, hex chars, traceid is 32 chars, version is `00`).
2. **No traceparent format validation.** Any string in the `traceparent`
   header — including obviously-malformed values like `"hello"` — will be
   accepted as a session id. W3C tracecontext explicitly says malformed
   traceparents must be ignored (treated as if no traceparent were present)
   so a downstream consumer can mint a fresh trace. Recommend validating
   shape before adopting the value.
3. **`tracestate` is silently ignored.** If a chain wants to propagate
   vendor-specific session metadata, `tracestate` is the conventional carrier;
   accepting `traceparent` but not `tracestate` is asymmetric. Probably out of
   scope for this PR but worth a follow-up.
4. **Lifetime of the session_id**: chaining LLM calls under one trace-id is
   exactly right for request-scoped chaining, but if the agent runs multiple
   user-facing turns inside a single distributed trace, all turns will collide
   on the same session id. Consider a docstring note explaining when this
   behavior is appropriate.

## Nits

- `or None` at the tail of the `or` chain on `:340` is dead — the previous
  `.get()` already returns `None` when missing. Drop it for clarity.
- The integration test on line 2336 imports `LiteLLMProxyRequestSetup` but
  the import is hidden inside a function in some of the new tests and at
  module level in others; pick one style.
- New tests use 4 different traceparent literals across the file. Define one
  module-level `EXAMPLE_TRACEPARENT = "00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01"`
  constant and reuse it.

## Sanitization

The traceparent values in tests
(`00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01`,
`00-abcdef1234567890abcdef1234567890-1234567890abcdef-00`) are W3C spec
example values, not secrets. The session id
`e96634a3-fa28-4083-b354-55542e2dca01` on line 2317 is a v4 UUID example, not
a real session.

## Risk

Medium. The stated intent ("chain calls that share a trace") is not what the
code actually does (it gives every span its own session). Land only after
extracting the trace-id portion of the traceparent and validating the header
shape.

## Recommendation

Request changes. Switch from "whole traceparent as session_id" to
"parse `version-traceid-spanid-flags`, validate, use traceid only as
session_id" — and update the tests to assert that two different traceparents
sharing the same trace-id collapse to the same session id.
