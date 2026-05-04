# BerriAI/litellm #27018 — [Fix] Isolate dual OTEL handlers

- Link: https://github.com/BerriAI/litellm/pull/27018
- Head SHA: (PR head on `BerriAI/litellm`)
- State: MERGED, +218/-18 across `litellm/integrations/opentelemetry.py` and the OTEL test file

## Summary
When two OTEL-based callbacks are configured simultaneously (e.g., a
"plain" `opentelemetry` exporter alongside `langfuse_otel`), they fight
over the global `TracerProvider` / `MeterProvider` / `LoggerProvider`,
which causes spans/metrics/logs to flow to the wrong backend. The
previous code special-cased only `langfuse_otel` for the tracer provider
and let it interfere everywhere else. This PR generalizes the escape
hatch via a new `OpenTelemetryConfig.skip_set_global` field plus a
helper `_skip_set_global()` that ORs that flag with the existing
`callback_name == "langfuse_otel"` check, and applies it to **all
three** provider types (tracer, meter, logger).

## Specific references
- `litellm/integrations/opentelemetry.py:71` — adds
  `skip_set_global: bool = False` to `OpenTelemetryConfig` with a
  one-line docstring.
- `litellm/integrations/opentelemetry.py:264-281` — `_get_or_create_provider`
  now branches: when `skip_set_global=True` and an existing SDK provider
  is found, log "skip_set_global=True; creating private %s for isolation"
  and call `create_new_provider_fn()` rather than reusing the global.
- `litellm/integrations/opentelemetry.py:303-308` — new
  `_skip_set_global(self) -> bool` consolidates the langfuse_otel
  special-case with the new config flag.
- `litellm/integrations/opentelemetry.py:319-336` — `_init_tracing` now
  passes `skip_set_global=self._skip_set_global()` and stashes the
  resolved provider on `self._tracer_provider`.
- `litellm/integrations/opentelemetry.py:362-365` — `_init_metrics` now
  also passes `skip_set_global=self._skip_set_global()` and stores
  `self._meter_provider`. Previously it always set the global.
- `litellm/integrations/opentelemetry.py:420-427` — `_init_logs` mirrors
  the same pattern, capturing the result into `self._logger_provider`.
- `litellm/integrations/opentelemetry.py:1098-1104` — the semantic-logs
  emitter now resolves its logger via `self._logger_provider.get_logger(...)`
  instead of the module-level `get_logger()` so that private providers
  actually receive the records.

## Notes
- The change correctly identifies that the previous tracer-only fix was
  necessary but not sufficient — meters and logs were still going through
  the global provider, breaking the same isolation guarantee.
- Switching `_emit_semantic_logs` to use the instance's logger provider
  is the linchpin. Without it, even a private LoggerProvider would emit
  through the global one.
- One subtle behavior change: the verbose-logger debug message changed
  from "Using existing %s" to "skip_set_global=True; creating private %s
  for isolation" when the flag is set. Operators grepping logs for the
  old message will need to update.
- The new test file additions (line 259+) appear to cover the
  langfuse-coexistence scenario; reviewer should confirm the new tests
  actually instantiate two handlers simultaneously and assert spans land
  in the right place.

## Verdict
`merge-as-is`
