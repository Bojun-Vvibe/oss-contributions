---
pr: google-gemini/gemini-cli#26218
sha: 072b6c46f78c1930a67e5e171da24d65f58784aa
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(cli): handle InvalidStream event gracefully without throwing

URL: https://github.com/google-gemini/gemini-cli/pull/26218
Files: `packages/cli/src/nonInteractiveCli.ts`,
`packages/cli/src/nonInteractiveCli.test.ts`
Diff: 114+/10-

## Context

Issue #24290: in non-interactive mode, when retries are exhausted on a
stream that yields `GeminiEventType.InvalidStream` (model returned an
empty response or a malformed tool call), the CLI fell off the bottom
of the event loop with no special handling. The result depended on the
output format: TEXT mode silently exited 0 with no output; STREAM_JSON
emitted a `result` event with `status: "success"` and an empty answer;
JSON mode produced a JSON envelope with empty content and no error
field. All three are user-hostile because the caller (script, CI,
operator at a terminal) gets no signal that the model actually failed
— they have to infer it from "exit 0 + empty stdout" which a wrapper
script will treat as success.

## What's good

- New `InvalidStream` arm in the event switch at `nonInteractiveCli.ts:399-413`:
  on hit, sets `invalidStreamError = "Invalid stream: The model returned
  an empty response or malformed tool call."`, then routes the message
  through the active output format —
  - STREAM_JSON: emits a `JsonStreamEventType.ERROR` event with
    `severity: "error"` and the message via `streamFormatter.emitEvent`,
  - TEXT: writes `[ERROR] ${message}\n` to stderr via
    `process.stderr.write`,
  - JSON: deferred to the final-output path (handled below).
  Then `toolCallRequests.length = 0; break;` — clears any accumulated
  tool calls (none of them can complete a turn the model failed to
  finish) and exits the inner stream loop, falling through to the
  finalisation path. The break instead of return is the right call so
  the per-format finaliser still runs for cleanup/metrics.
- The finaliser path at `:524-538` is the load-bearing companion change:
  - STREAM_JSON `result` event status flips from hard-coded `"success"`
    to `invalidStreamError ? "error" : "success"`, so the `result`
    envelope downstream consumers parse for status accurately reflects
    the failure.
  - JSON formatter at `:531-538` now passes a fourth argument
    `invalidStreamError ? { type: "INVALID_STREAM", message:
    invalidStreamError } : undefined` to `formatter.format(...)`, so
    the JSON envelope grows an `error: { type, message }` field
    visible in the test assertion at `nonInteractiveCli.test.ts:2092-2095`
    (`output.toContain('"error": {')`,
    `output.toContain('"type": "INVALID_STREAM"')`).
  - TEXT mode falls through to `ensureTrailingNewline()` and exits with
    no extra output, since the stderr write already happened in the
    event arm — no double-print.
- Test coverage at `nonInteractiveCli.test.ts:2040-2098` is one test
  per output format, each asserting (a) the right error surface
  (stderr `[ERROR] ...`, JSON-stream `"type":"error","severity":"error"`,
  JSON envelope `"type": "INVALID_STREAM"`), and (b)
  `mockGeminiClient.sendMessageStream).toHaveBeenCalledTimes(1)` — i.e.
  no retry past the InvalidStream event, which is correct because by
  contract `InvalidStream` is emitted only after the upstream stream
  layer has already exhausted its retry budget.
- The `vi.mocked(...)` → `vi.spyOn(...)` migrations at `:706, :796, :839,
  :1533, :1695, :1870, :1934, :2299` are noisy but mechanically correct
  for sites that need to mock `uiTelemetryService.getMetrics` —
  `vi.mocked()` requires the source to be a `vi.fn()` already (which it
  is in the global setup) but `vi.spyOn` is the more robust pattern
  that doesn't depend on hoisting order, and the new tests at `:2050,
  :2074` use `vi.spyOn` directly on `mockConfig.getOutputFormat` to
  switch format per-test. The mass-rename keeps the file consistent
  rather than mixing two mocking idioms.
- Message string at `:401` ("Invalid stream: The model returned an empty
  response or malformed tool call.") is reused across all three output
  paths and the three tests assert the same substring (`:2058`,
  `:2080-2082`, `:2094-2095`) — a single source of truth means a future
  message tweak only needs one diff site.

## Verdict reasoning

Closes a real "silent success on model failure" UX bug across all three
non-interactive output formats with a uniform shape — every format now
emits a discoverable error signal (stderr line, JSON-stream error
event, JSON envelope error field), the `result.status` correctly
reflects the failure for stream consumers, and the test matrix locks
the contract per format with the right assertions
(no-retry-past-InvalidStream + format-specific error surface visible).
Mocking-pattern cleanup is a wash for risk but improves consistency.
The kind of fix that should land same-day to stop scripted callers from
treating empty-output exits as success.
