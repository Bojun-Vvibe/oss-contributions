# google-gemini/gemini-cli PR #26392 — fix(windows): Resolve hangs, zombie processes, and improve subagent reliability

- **Repo:** google-gemini/gemini-cli
- **PR:** #26392
- **Head SHA:** `4ff5a60f159b0cc6b3a1e1bc1edc304169b8d27e`
- **Author:** DovahkiinYuzuko
- **Title:** fix(windows): Resolve hangs, zombie processes, and improve subagent reliability
- **Diff size:** +170 / -49 across 9 files
- **Drip:** drip-295

## Files changed

- `packages/cli/src/commands/gemma/setup.ts` (+49/-32) — wraps the model download `fetch(...)` in an `AbortController` with a hardcoded `setTimeout(() => controller.abort(), 30000)`. Re-throws as `"Download failed: Connection timeout (30s)"` on `AbortError`.
- `packages/cli/src/ui/components/LoadingIndicator.tsx` (+1/-1) — `'Thinking...'` becomes `` `Thinking (${elapsedTime}s)...` ``.
- `packages/cli/src/ui/utils/commandUtils.ts` (+4/-3) — `isSlashCommand` now `.trim()`s the query before `.startsWith('/')` checks.
- `packages/cli/src/utils/activityLogger.ts` (+33/-3) — `bufferLimit` raised from 10 to 50; new `flushConsoleBuffer()` writes to `~/.gemini/logs/latest.log` with a 512KB rotation that trims to the last 500 lines. `logConsole` switches from "shift on overflow" to "flush on threshold reached".
- `packages/core/src/agents/local-executor.ts` (+23/-0) — `LocalAgentExecutor` adds `consecutiveErrorCount` tracking; if 3 consecutive turns have all tool responses failing, sets `terminateReason = AgentTerminateMode.ERROR` and breaks.
- `packages/core/src/agents/local-invocation.ts` (+5/-2) — subagent failure path now sets `error: { message, type: ToolErrorType.EXECUTION_FAILED }` (was previously omitted intentionally to preserve rich `returnDisplay`).
- `packages/core/src/ide/process-utils.ts` (+2/-0) — adds `timeout: 5000` to the powershell `Get-CimInstance Win32_Process` `execAsync` call.
- `packages/core/src/services/executionLifecycleService.ts` (+34/-1) — on `kill(executionId)`, on Windows additionally runs `spawnSync('taskkill', ['/F', '/T', '/PID', executionId.toString()])` for both the "execution not found" path and the post-`execution.kill?.()` path.
- `packages/core/src/tools/write-file.ts` (+19/-7) — `getCorrectedFileContent` wraps `ensureCorrectFileContent` in `try/catch` and falls back to `proposedContent` on either thrown error or empty-string result.

## Specific observations

- This PR bundles **eight independent fixes** under one Windows-themed title. Several aren't Windows-specific (the `LoadingIndicator` text change, `isSlashCommand` trim, `activityLogger` flush behavior, the consecutive-tool-error breaker in `local-executor`, the `write-file` fallback). That makes it hard to bisect, hard to review, and hard to revert. Strong recommendation: split into a Windows-only PR (`process-utils` timeout, `executionLifecycleService` taskkill, possibly the `setup.ts` download timeout) and per-feature PRs for the rest. The maintainers will likely ask for this anyway.
- `setup.ts:18-19` — `setTimeout(() => controller.abort(), 30000)` with a hardcoded 30s timeout. Model downloads can be hundreds of MB on slow connections; 30s is **too aggressive for a download timeout**. This should either be (a) a connection/idle timeout (reset on each `reader.read()` chunk), or (b) configurable via env var. As written, anyone on a slow connection or downloading a large gemma model gets a confusing failure 30s in.
- `setup.ts:94-95` — `clearTimeout(timeoutId)` in `finally` is correct. The `try/catch/finally` shape is fine.
- `executionLifecycleService.ts:574-590` — `taskkill /F /T` is a force-kill that recursively kills the process tree. That's the right tool, but the call uses `executionId.toString()` directly — `executionId` is checked against `NON_PROCESS_EXECUTION_ID_START` but there's no validation it's actually a real OS PID. If `executionId` is somehow `0` or negative, you risk `taskkill /F /T /PID 0` which on Windows tries to kill the System Idle Process. Add an `executionId > 0` guard.
- `executionLifecycleService.ts:574-590` — the "execution not found" branch additionally runs taskkill. That's a behavior change: previously a kill-after-completion was a no-op; now it's an external process call to taskkill that fires on every double-kill. Cheap but observable in logs. Worth adding a one-line comment.
- `process-utils.ts:282-285` — `timeout: 5000` on the powershell `Get-CimInstance` call. Five seconds is reasonable for a process listing on most machines. On a busy Windows box with thousands of processes the WMI call can legitimately exceed 5s; on timeout `execAsync` will reject and the caller will see no process info at all. The comment correctly notes "5s timeout to prevent multi-minute hangs", but consider degrading gracefully: catch the timeout and return an empty `Map` rather than letting the error propagate.
- `local-executor.ts:677-697` — the "3 consecutive failed turns ⇒ terminate with ERROR" heuristic is a useful safety net, but the failure detection is fragile: `p.text?.includes('failed or were unauthorized')` is a string-substring check on a tool response. Any future tool that legitimately echoes that phrase (eg. a retrospective summarizer) will trigger the breaker. Use a structured signal — every `functionResponse` already carries `response.error`; rely solely on that, not on a text substring. The threshold of 3 should also be a constant with a name.
- `local-invocation.ts:374-381` — the prior comment explicitly said "We omit the 'error' property so that the UI renders our rich returnDisplay". This PR removes that comment and adds the `error` property. That **may regress the UI** — confirm with the maintainers whether the rich `returnDisplay` is still preferred or whether the structured error is now the contract.
- `activityLogger.ts:148-187` — `flushConsoleBuffer()` uses synchronous `fs.appendFileSync` / `fs.statSync` / `fs.readFileSync` / `fs.writeFileSync` on the hot path of `logConsole`. For a 512KB file this is fast on SSD but blocks the event loop. Switch to `fs.promises.*` and `await this.flushConsoleBuffer()`. The `void this.flushConsoleBuffer()` at line 203 also discards the returned promise — any rejection becomes an unhandled rejection.
- `activityLogger.ts:179-186` — the rotation reads the whole file into memory just to keep the last 500 lines. Use a streaming tail or rotate by file rename (`latest.log` → `latest.1.log`). Current approach also has a TOCTOU race between `statSync`, `readFileSync`, and `writeFileSync` if multiple processes share the file.
- `write-file.ts:363-381` — silent fallback to `proposedContent` on AI-correction failure is conservative and probably correct, but `debugLogger.warn` will be invisible to most users. Consider surfacing this once per session via a one-time toast or status line, otherwise users will be confused why a regression in the LLM-correction path is invisible.
- `LoadingIndicator.tsx:71` — `Thinking (${elapsedTime}s)...` is a UX improvement. No issue.
- `commandUtils.ts:42-50` — trimming before `startsWith('/')` is correct for the leading-whitespace case but changes the contract subtly: previously `" /help"` was not a slash command; now it is. If the prompt accepts file paths or other inputs that may legitimately start with whitespace+`/`, this is a behavior change that could route a non-command to the slash dispatcher.

## Verdict: `request-changes`

The Windows-specific pieces (`process-utils.ts` timeout, `executionLifecycleService.ts` taskkill) are valuable and roughly correct, modulo PID validation. But the PR as a whole has three structural issues: (1) it bundles 8 unrelated changes under one title — split it; (2) the 30s download timeout in `setup.ts` is hardcoded and aggressive enough to break legitimate large-model downloads; (3) the `local-executor` consecutive-failure breaker uses fragile text-substring matching against `'failed or were unauthorized'` and should use structured error signals. The `activityLogger` sync I/O on the hot path is also worth fixing before merge. The `local-invocation` UI-error contract change should be confirmed with the maintainers, not assumed.
