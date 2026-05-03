# QwenLM/qwen-code PR #3779 — feat(telemetry): per-signal HTTP OTLP endpoints + log→span bridge

- Link: https://github.com/QwenLM/qwen-code/pull/3779
- SHA: `3a0d16e247f7469b31f48a9af4d07801c8a0a443`
- Author: doudouOUC
- Stats: +1387 / −102, multiple files (docs, cli/gemini.test.tsx, telemetry config + new `LogToSpanProcessor`)

## Summary

Adds explicit HTTP-OTLP signal routing (`resolveHttpOtlpUrl()` appends `/v1/traces`, `/v1/logs`, `/v1/metrics` per the OTel spec, preserving query strings) plus per-signal endpoint overrides (`otlpTracesEndpoint`, `otlpLogsEndpoint`, `otlpMetricsEndpoint`) for backends with non-standard paths (e.g. Alibaba Cloud's `/api/otlp/traces`). Also introduces a `LogToSpanProcessor` that bridges OTel log records to spans for traces-only backends, deriving a stable 128-bit traceId from `SHA-256(sessionId)`. Auto-wires the bridge when traces URL exists but logs URL doesn't. Closes #3734.

## Specific references

- `docs/developers/development/telemetry.md` L58–L82: docs table is updated with all three new per-signal settings and the "HTTP OTLP signal routing" note explicitly documents both behaviours: append-default and per-signal override. Importantly it also calls out the standard `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` env-var fallback and clarifies precedence (`QWEN_TELEMETRY_OTLP_*` > `OTEL_*`). Good — most projects forget to spell out the precedence rule.
- L107–L142: the Alibaba-Cloud "Option B" example uses `http://<host>/<token>/api/otlp/traces` literally. Worth noting in docs that putting the token in the URL path is one valid auth shape but headers (`OTEL_EXPORTER_OTLP_HEADERS`) is the other; the doc already mentions headers in the next paragraph, so this is consistent.
- `packages/cli/src/gemini.test.tsx` L493–L530: `initialSigintListeners` / `initialSigtermListeners` are now snapshotted in `beforeEach` and restored in `afterEach`. Reasonable hardening — the new telemetry bridge probably installs its own signal handlers for graceful shutdown, and without the snapshot/restore you would leak listeners across tests. The cleanup loop iterates `process.listeners('SIGINT')` and removes anything not in the initial set; this is the standard Node test-isolation idiom.
- `LogToSpanProcessor` (per PR body): traceId derivation is `SHA-256(sessionId)` truncated to 128 bits. SHA-256 is overkill cryptographically for an ID derivation, but it's deterministic and uniformly distributed, and 128 bits is the W3C TraceContext requirement, so this is fine. Make sure the implementation also sets `spanId` (TraceContext requires both) — not visible in the diff hunk shown.
- Auto-wire rule "traces URL exists but logs URL doesn't" → install bridge: this is the correct heuristic for traces-only backends, but it could surprise users who deliberately disabled logs. A one-line warning log on auto-install ("Logs endpoint not configured; bridging logs into traces via LogToSpanProcessor") would make it discoverable.
- Test plan in PR body is exhaustive (unit tests for `resolveHttpOtlpUrl` covering base URL, trailing slash, explicit signal path, custom prefix, HTTPS, query strings; unit tests for `LogToSpanProcessor` covering attribute mapping, duration, traceId derivation, error status, flush/shutdown; integration tests for per-signal overrides; env-var resolution tests; round-trip tests for new TelemetrySettings fields). The 664+ existing tests claimed to still pass.

## Verdict

verdict: merge-after-nits

## Reasoning

The OTel spec compliance is correct (`/v1/<signal>` append, query-string preservation, per-signal override precedence) and the documentation actually explains both the default behaviour and the override mechanism, which is rare. The `LogToSpanProcessor` is a pragmatic answer to traces-only backends. Nits before merge: (1) emit an info-level log when the bridge auto-wires, so the behaviour is discoverable; (2) confirm `spanId` is generated alongside the derived traceId; (3) consider exposing the per-signal endpoints as CLI flags too (the table marks them `-` for CLI flag, which is an asymmetry with `otlpEndpoint`). None blocking.
