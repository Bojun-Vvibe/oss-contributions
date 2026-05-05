# google-gemini/gemini-cli #26504 — fix(cli): provide JSON output for AgentExecutionStopped in non-interactive mode

- **PR:** google-gemini/gemini-cli#26504
- **Head SHA:** `3708f88ea704b1f8218760cf5598f0a86b9e64ad`
- **Files:** 3 changed (+155 / -0) — pure additive, two test files plus one prod-code branch addition

## What changed

- `packages/cli/src/nonInteractiveCli.ts:403-416` — the `AgentExecutionStopped` event handler grows two new branches. When the configured output format is `OutputFormat.JSON`, it now instantiates a `JsonFormatter`, pulls metrics from `uiTelemetryService.getMetrics()`, and writes a fully-formed JSON envelope with `session_id`, `responseText`, `stats`, and `[...warnings, stopMessage]` (so the stop reason becomes a synthetic warning rather than being silently dropped). The new `else` branch calls `textOutput.ensureTrailingNewline()` for the text-format fallback. Previously the handler only emitted on `STREAM_JSON` and produced nothing on `JSON` mode — a hook-stop in `--output-format json` was returning an empty payload to the caller.
- `packages/cli/src/nonInteractiveCli.test.ts:2048-2117` adds two regression tests: one asserts that on `OutputFormat.JSON` with an `AgentExecutionStopped` event carrying `reason: "Stopped by hook"`, `process.stdout` receives a stringified envelope containing `warnings: ['Agent execution stopped: Stopped by hook']`; the other asserts `STREAM_JSON` mode emits a `result` event with `status: "success"`.
- `packages/cli/src/nonInteractiveCliAgentSession.test.ts:2210-2280` mirrors both tests for the agent-session non-interactive path, with the small but meaningful divergence that the session-test JSON envelope does **not** include the `warnings` field — the agent-session formatter signature gets `undefined` for that arg in the new code path. Worth confirming this asymmetry is intentional (the user-visible reason for the stop wouldn't surface in agent-session JSON output).

## Risks / notes

- The status reported in `STREAM_JSON` is `"success"`, not `"stopped"` or `"interrupted"`. A hook actively halting the agent being reported to scripts as `success: true` is a semantic stretch — the partial response did succeed, but the run was cut short by policy. Downstream automation that branches on `status` may now see hook-blocked runs as indistinguishable from clean completion. A `status: "stopped"` (or at least a top-level `stop_reason` field) would make this much more useful for callers.
- In the JSON-mode envelope (not stream-JSON), the `responseText` field carries `'Partial content'` — the text emitted before the stop event — but there's no signal that the response is truncated. Combining this with the `success` status above means a script reading just `response` has no way to know it's not the full answer.
- The asymmetric `warnings`/no-`warnings` treatment between `nonInteractiveCli` and `nonInteractiveCliAgentSession` looks like a copy-paste regression in the agent-session formatter call (passes `undefined` instead of `[stopMessage]`). If agent-session JSON output is supposed to surface stop reasons, that's a bug; if it's intentionally quieter, document it.
- No new prod code in `nonInteractiveCliAgentSession.ts` itself appears in the diff window — only its tests. Worth confirming the agent-session path actually has equivalent JSON-mode handling and the tests aren't asserting against an existing-but-broken behaviour.

## Verdict

**request-changes** — the JSON envelope shape needs a real "stopped" signal (either `status: "stopped"` or a top-level `stop_reason` field), and the asymmetry between `nonInteractiveCli` (warnings include stop) and `nonInteractiveCliAgentSession` (no warnings) needs to be reconciled or explicitly documented as intentional.
