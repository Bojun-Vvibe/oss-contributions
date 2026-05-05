# openai/codex #21107 — Avoid noisy OTEL diagnostics in codex exec

- URL: https://github.com/openai/codex/pull/21107
- Head SHA: `c743089d83925ca7810cbc766c3b457a7188c99b`
- Author: cpaasch-oai
- Size: +106 / -10 across 3+ files

## Comments

1. `codex-rs/core/src/config/mod.rs:3173-3179` — Changing `let trace_exporter = t.trace_exporter.unwrap_or_else(|| exporter.clone())` to `let trace_exporter = t.trace_exporter.unwrap_or(OtelExporterKind::None)` is a **behavior change**, not just noise reduction. Before: setting only `[otel] exporter = ...` would also enable trace export to the same endpoint. After: traces are off unless explicitly opted in. The inline comment explains why ("OTLP HTTP endpoints are signal-specific in our config, so enabling log export must not implicitly send spans to a /v1/logs endpoint") and that's a defensible reason — sending OTLP spans to a `/v1/logs` URL just produces 400s from the collector, which is exactly the noise this PR is trying to silence. Good fix, but it's a config-default break for anyone whose existing setup relied on the cloning behavior. **Needs a CHANGELOG note** flagged as a breaking-default change.
2. `codex-rs/core/src/config/config_tests.rs:6489-6519` — `trace_exporter_defaults_to_none_when_log_exporter_is_set` is exactly the right regression test for the change above. Asserts both that the log exporter is preserved (`OtelExporterKind::OtlpHttp { .. }`) and that the trace exporter is forced to `None`. Good naming, good shape.
3. `codex-rs/core/src/config/config_tests.rs:6486` — Worth adding a paired test: `trace_exporter_uses_explicit_value_when_set` to lock in that an explicit `[otel] trace_exporter = "otlp-http"` *still* works. The current test only covers the implicit-default path; without the paired test, a future "simplify by always forcing trace_exporter to None" refactor wouldn't be caught.
4. `codex-rs/exec/src/lib.rs:156` — `EXEC_DEFAULT_LOG_FILTER = "error,opentelemetry_sdk=off,opentelemetry_otlp=off"` — `error` as the global default is the right level for a headless `codex exec` invocation. Targeted `=off` for the two opentelemetry crates surgically silences the spam. Worth adding `opentelemetry=off` as well in case future versions emit at the umbrella crate level.
5. `codex-rs/exec/src/lib.rs:224-232` — `exec_stderr_env_filter()` honors `RUST_LOG` first (`try_from_default_env`) and only falls back to the silent default. Correct precedence — operators who *want* the OTEL diagnostics for debugging can `RUST_LOG=opentelemetry_otlp=debug codex exec ...` and get them. The comment ("OTEL export is best-effort; keep exporter self-diagnostics out of headless command output unless the caller opts in with RUST_LOG") perfectly explains the design decision. Excellent.
6. Scope concern: this PR bundles two logically separate changes — (a) a config-default change to `trace_exporter`, (b) a log-filter change to `codex exec`. They're related (both reduce OTEL noise) but a future bisect for "why are my traces not being exported anymore?" would be easier if (a) lived in its own commit. Not a blocker — both are well-justified — just a hygiene note.

## Verdict

`merge-after-nits`

## Reasoning

Both changes are correct and well-justified. The trace_exporter default flip is a sensible breaking change (sending spans to a logs endpoint never worked) and is gated behind explicit user config. The exec log filter is exactly the right scope — silences noise by default, opt-in via `RUST_LOG`. Two asks: (1) CHANGELOG entry flagging the trace_exporter default change for users with existing OTEL configs; (2) one more test asserting that an explicit `trace_exporter` value still wins. The split-into-two-commits suggestion is optional.
