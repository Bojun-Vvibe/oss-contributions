# sst/opencode PR #24883 — feat: add trace for whole workflow

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/24883
- Head SHA: `b9831463498725d5d3895aeb896bed45f02dd7fd`
- State: OPEN, +855/-14 across 16 files

## What it does

Introduces a server-side **workflow trace**: per-HTTP-request structured JSON dropped into `~/.local/share/opencode/trace/trace_<request_id>_<iso>.json` (configurable via new `Global.Path.trace`). Captures HTTP metadata, an `events` timeline, and a `chat` block (`user_input`, `model_stream_text`, `part_deltas`, `assistant_output`, `model`). Threads the trace session through Effect via a new `WorkflowTraceSessionRef` (Async-Local-Storage can't traverse Effect runtime boundaries). Defaults: writes only for `POST /session/:id/{message,prompt_async,command,shell}`; widened by `OPENCODE_WORKFLOW_TRACE=all` env or `x-opencode-workflow-trace: 1` header.

Notably also ships a self-postmortem in `WORKFLOW-TRACE-PERF.md` documenting an O(n²) string-concat regression caught during dev: `model_stream_text = model_stream_text + delta` and `_traceTextBuf += delta; slice(...)` on every stream event were saturating the event loop. Fix: `string[].push` + single `join`, plus deferred `setImmediate` for `materialize + JSON.stringify + fs.write` so the HTTP request path doesn't block on persistence.

## Specific reads

- `packages/opencode/src/server/workflow-trace.ts` (new, the heart of the feature) — `appendModelStreamText`, `traceRecordMessagePartDelta`, `scheduleWorkflowTracePersist` are the three load-bearing primitives. The `scheduleWorkflowTracePersist` deferred-write pattern (`setImmediate(async () => { await materialize(); await writeTraceFile(...); })`) is exactly the right primitive for "persist after response sent, don't block the finally chain". Worth a comment that the `setImmediate` callback is fire-and-forget — any thrown error in `materialize` becomes an unhandledRejection unless wrapped.
- `packages/opencode/src/server/routes/instance/session.ts` — `prompt_async` 204-then-background path defers `scheduleWorkflowTracePersist` to `runRequest.finally` so the persisted trace contains the *actual* model reply, not just `user_input`. This is correct and called out in the PR body; pin it with a regression test that asserts `assistant_output` is non-empty for a `prompt_async` trace.
- `packages/opencode/src/session/processor.ts` — every stream event still calls `traceRecordLlmStreamEvent`. The PR claims this is now O(1) amortized via the chunk-array pattern, but the call site itself runs on the hot stream-event path. Pin a perf bound: a microbenchmark that asserts trace-disabled vs trace-enabled `processor.ts` event throughput stays within, say, 10% — the postmortem documents the concrete prior failure mode (global event-loop saturation), so a regression guard is cheap insurance.
- `packages/core/src/global.ts` — adds `Path.trace`. Confirm the directory is created with `recursive: true` and a `0o700` mode (trace files contain raw user prompts and model outputs — privacy-sensitive). The PR body lists `OPENCODE_TRACE_MAX_USER_CHARS` / `OPENCODE_TRACE_MAX_STREAM_CHARS` / `OPENCODE_TRACE_MAX_ASSISTANT_CHARS` / `OPENCODE_TRACE_MAX_PART_DELTAS` / `OPENCODE_TRACE_PART_DELTA_CAP` env knobs, which is the right shape, but no per-file size cap or per-day disk-usage cap — a long-running daemon with `OPENCODE_WORKFLOW_TRACE=all` will accumulate files indefinitely.
- `WORKFLOW-TRACE-PERF.md` (new, root) — postmortem doc with self-check list ("growing-string `+=` in hot path?", "large `JSON.stringify` synchronous before request returns?", "disk I/O on HTTP `finally` critical path?"). This is genuinely useful onboarding artifact and the right place to document the design-correction story (`llm_stream` → `part_deltas` realignment with UI bus granularity). The reflection section ("观测与产品同构") is the principle worth keeping.
- `changelog/20260428/readme.md` — milestone changelog with cross-links to the touched files. Good practice; no PR-side issue.

## Risk

- 855 LOC across 16 files for a feature that started as one shape (`llm_stream`) and pivoted to another (`part_deltas`) mid-implementation. The PR body acknowledges this honestly. Risk is that residual `llm_stream`-shaped code paths still register traces (the postmortem says `traceRecordLlmStreamEvent` is "kept" — confirm whether it's a backward-compat alias or live duplicate code that doubles work).
- Default-on for the four `POST /session/:id/...` routes means every interactive prompt now writes a JSON file. Cumulative disk growth with no GC story. Operators on small disks (or inside containers) will need a manual cleanup job. A `OPENCODE_TRACE_MAX_FILES` or `OPENCODE_TRACE_RETENTION_DAYS` env, or a `Path.trace` rotation hook, closes that gap.
- `WorkflowTraceSessionRef` adds an Effect `Ref` that's read on every `traceRecordLlmStreamEvent`. Confirm reads are zero-allocation (Effect `Ref.get` typically is) — but worth pinning in the same microbenchmark.

## Verdict

`merge-after-nits` — feature is well-motivated, the postmortem is exemplary engineering hygiene, and the design-pivot honesty is rare and welcome. Four nits: (1) wrap the `setImmediate` callback in `try/catch` (or use `void scheduleWorkflowTracePersist().catch(log.warn)`) to avoid unhandled-rejection process exits, (2) add per-day or per-file-count rotation/retention so the trace dir doesn't grow unbounded, (3) ensure `Path.trace` is created with `0o700` since traces contain raw prompts, (4) add a microbenchmark pinning the no-trace vs trace-enabled `processor.ts` stream-event throughput delta so the O(n²) regression class is guarded against future refactors.
