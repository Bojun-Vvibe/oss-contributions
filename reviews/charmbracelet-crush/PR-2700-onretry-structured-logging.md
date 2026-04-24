# PR-2700 — fix(agent): implement OnRetry logging with structured retry fields

[charmbracelet/crush#2700](https://github.com/charmbracelet/crush/pull/2700)

## Context

`internal/agent/agent.go` had a `// TODO: implement` body inside the
`OnRetry: func(err *fantasy.ProviderError, delay time.Duration)`
callback registered on the agent's stream loop. Provider retries
(429s, 5xx, transient TLS errors) were happening silently — no log
line, no metric, no operator visibility. This PR fills the body with
a single `slog.Warn("Provider request failed, retrying", ...)` call
fed by a new helper `providerRetryLogFields(err, delay)` that builds
a structured `[]any` of `retry_delay`, `status_code`, optional
`title`, optional `message`. Three table-driven tests cover nil err,
fully-populated err, and status-only err.

PR description is explicit: "No change to retry policy, retry count,
retry delay, or control flow. This is observability-only behavior."

## Strengths

- **Behavior-neutral by design.** The `OnRetry` callback already
  fires from the underlying provider library; only the body changes.
  Zero risk of altering retry semantics.
- **Helper is testable in isolation.** `providerRetryLogFields`
  returns the field slice rather than calling `slog.Warn` directly.
  Tests assert exact slice contents — that's the right granularity
  for a log-shape contract you don't want to drift.
- **Optional fields are conditional.** Empty `title`/`message`
  aren't logged as empty strings; the slice stays compact. Good for
  log-volume reasons and for log-aggregation tools that show every
  field.
- **`retry_delay` is logged as `delay.String()`** (e.g. `"1.5s"`)
  rather than nanoseconds. Human-readable.
- Tiny diff, easy to review, easy to revert.

## Concerns / risks

- **No attempt counter.** The callback signature has only `err` and
  `delay` — but operators investigating "why is this session
  burning quota?" want to know whether they're seeing one retry or
  the 5th. If the provider library exposes an attempt index
  anywhere (callback args, context value), thread it through. If
  not, this is an upstream `fantasy` API gap worth filing.
- **No retry counter / metric.** A `slog.Warn` is fine for
  debugging, but at scale you want a counter (`crush_provider_retry_
  total{status="429"}`) so dashboards and alerts can fire on retry
  storms. Logging is necessary but not sufficient. Worth at least a
  `// TODO: emit metric` so the observability work doesn't stop at
  the log line.
- **`status_code` is logged as an `int` field name** but
  `fantasy.ProviderError.StatusCode` may be 0 for non-HTTP failures
  (TLS handshake error, DNS failure). Logging `status_code: 0` is
  noisier-than-useful. Consider conditionally including it only
  when nonzero, mirroring the title/message handling.
- **`message` may include the full upstream response body** for
  some providers, which can contain a request ID and rarely a
  prompt echo. At debug verbosity this is fine; at warn level on a
  shared stderr it could leak prompt content into log files.
  Consider truncating to N chars or moving the body to a debug
  field.
- **No log when retry is *exhausted***. Operators see "retrying" N
  times but no terminal "gave up" event from this layer — that
  signal probably lives elsewhere, but worth a one-line comment
  pointing the next reader at it.
- **Test asserts exact field order.** `require.Equal(t, []any{
  "retry_delay", "2s"}, fields)` — if a maintainer reorders fields
  for readability (e.g. `status_code` first), the test fails even
  though the log output is semantically identical. Consider a map-
  comparison helper if the order isn't actually contractual.

## Verdict

**Approve.** Right-sized fix for an observability gap. Follow-ups
(file these as separate issues, don't block on them): plumb a retry
attempt index through the `fantasy.OnRetry` signature; add a counter
metric alongside the log line; conditionally elide `status_code: 0`;
decide whether `message` content should be truncated.
