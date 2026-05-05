# QwenLM/qwen-code #3847 — feat(telemetry): inject traceId/spanId into debug log files for OTel correlation

- **Head:** `68b5a90` (`68b5a90f6ab56fa9e037c8148134249f20ce47f9`)
- **Repo:** QwenLM/qwen-code
- **Scope:** Two main attack surfaces:
  1. `packages/core/src/config/config.ts:74, 1380` — imports + invokes new `refreshSessionContext(this.sessionId)` in the session refresh path
  2. `packages/core/src/core/client.ts:95, 1120-1192` — wraps `generateContent()` body in a `withSpan('client.generateContent', { model, prompt_id }, async () => { ... })` block
  3. `packages/core/src/core/coreToolScheduler.ts:78-79, 1759-1760` — wraps the per-tool execution body in a `withSpan(`tool.${toolName}`, { tool_name, call_id }, async (span) => { ... })` and threads `span` so hook-blocked / hook-stopped paths can call `span.setStatus({ code: SpanStatusCode.ERROR, ... })`

(Diff cut at ~400 lines but the pattern is consistent — `withSpan` wrappers around units of work that already had structured logging, plus a session-context refresh hook on session-id rotation so the sessionId matches the OTel resource attribute on logs.)

## What this enables

Without spans on the critical-path async functions (LLM call, tool execution), the existing OTel log exporter in qwen-code emits log records with no `trace_id` / `span_id` correlation IDs, so log lines can't be joined to traces in Jaeger / Tempo / Honeycomb / etc. This PR closes that gap by establishing the span context at the boundary of `generateContent` and per-`tool.<name>` execution so the existing debug-log-to-OTel pipeline picks up the active span via `context.active()`.

## Strengths

- **`withSpan` is the right abstraction** — pre-existing in the telemetry module, already handles span end on both resolve and reject paths, so wrapping is a low-risk transform.
- **Span attribute keys are sensible**: `model`, `prompt_id`, `tool_name`, `call_id` are exactly the fields a developer searches by when triaging a failed run.
- **Hook-blocked path correctly sets `SpanStatusCode.ERROR`** at the `preHookResult.shouldProceed === false` branch — without that, hook-blocked tool executions would show as `OK` spans, masking real prod issues.
- **`refreshSessionContext(this.sessionId)` at `config.ts:1380`** is the right place: the comment immediately above clarifies that prototype-overridden Configs (those built via `Object.create`) call their own clear path, so the parent's session-context refresh doesn't double-fire.

## Concerns

1. **Span attribute `prompt_id: promptIdOverride ?? ''`** at `client.ts:1126` will record an empty-string attribute when `promptIdOverride` is undefined. The actual prompt id used inside the function is resolved later via `promptIdContext.getStore() ?? this.lastPromptId!`. The span attribute should reflect the *resolved* prompt id, not the override-or-empty. Easy fix: hoist the resolution out, set the attr from the resolved value (or via `span.setAttribute('prompt_id', resolved)` inside the lambda).
2. **`tool.${toolName}` span name cardinality** at `coreToolScheduler.ts:1762`. Tool names are bounded (built-in set + finite MCP tools), so this is fine for OTel backends. But if a future MCP server publishes high-cardinality tool names (e.g., `query_table_<uuid>`), this could blow up backend cardinality budgets. Worth a one-line note in the OTel docs that tool names are span-name dimensions.
3. **Diff is large and bundles two concerns** — the `refreshSessionContext` work and the span-instrumentation work are independently testable but ship together. Splitting would make reviewer's job easier; not a blocker.
4. **No tests visible in the read window** — `withSpan` wrapping is hard to assert without a span exporter mock, but the existing `tracer.ts` should already have one. At least one test that asserts a `tool.read` span is recorded with `tool_name: "read"` would prevent regression.
5. **`SpanStatusCode.ERROR` is set on hook-block at `:1820` but the comparable post-hook `shouldStop` branch around `:1855-1872` (visible in the read window) also sets `setStatus` — verify all error returns inside the wrapped lambda call `setStatus` consistently.** The diff window cuts before showing the success path, but if the success path doesn't explicitly set `OK` it will default to UNSET which is fine per OTel semantics.
6. **Brand-name compliance** — all references in the diff use `qwen-code` / `Qwen Code`, consistent with the rest of the project.

## Verdict

**merge-after-nits** — fix the `prompt_id: ... ?? ''` attribute at `client.ts:1126` to use the resolved id, add at least one OTel-trace assertion test, and consider splitting the session-context-refresh from the span-wrap commit. The instrumentation itself is the right shape and unblocks real OTel-correlated debugging. Head `68b5a90`.
