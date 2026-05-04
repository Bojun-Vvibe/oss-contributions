# sst/opencode#25739 — fix(observability): respect OTEL_SERVICE_NAME and service.name attribute

- PR ref: `sst/opencode#25739` (https://github.com/sst/opencode/pull/25739)
- Head SHA: `de9387e5fa98ef29bdc5209ef48226b1e35491de`
- Title: fix(observability): respect OTEL_SERVICE_NAME and service.name attribute
- Verdict: **merge-as-is**

## Review

Tiny, correct, single-file diff in `packages/core/src/effect/observability.ts:39-52`.
The precedence order — `OTEL_SERVICE_NAME` env var > caller-supplied
`attributes["service.name"]` > literal `"opencode"` fallback — matches the OTel
SDK spec verbatim (the env var is the documented override, and the resource
attribute is the documented schema field). Hard-coding `"opencode"` as the
serviceName string regardless of either input was a real bug for anyone shipping
spans into a multi-tenant collector.

The destructure trick at line 43 (`const { "service.name": _removed,
...remainingAttributes } = attributes`) is a clean way to strip the now-promoted
`service.name` from the spread that follows so it doesn't double-write. The
prefixed `_removed` name correctly tells the linter to ignore the unused binding.

No tests were added, but for a 5-line precedence change inside a `resource()` builder
that already returns a plain object, an assertion that
`OTEL_SERVICE_NAME=foo` wins over `attributes["service.name"] = "bar"` is
arguably overkill — the diff is its own proof. Ship it as-is.
