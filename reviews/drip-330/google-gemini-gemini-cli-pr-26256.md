# google-gemini/gemini-cli #26256 — fix(shell): stop foreground commands after excessive output

- SHA: `886b0afe8e74b11af6c65a26e1d3b82bc8f4db37`
- State: OPEN, +226/-4 across 6 files
- Related: #14367, #25615

## Summary

Adds a 10 MiB foreground-output guard for the shell tool: when a foreground command exceeds `maxOutputBytes` (default `10 * 1024 * 1024`), the process group is SIGTERM'd, the result is annotated with `outputLimitExceeded: true`, and a `[GEMINI_CLI_WARNING: …]` message in `result.output` redirects the model to background mode + `read_background_output`. Backgrounded commands disable the guard. Covers both the PTY path and the `child_process` fallback with parallel test cases.

## Notes

- `packages/core/src/services/shellExecutionService.ts:50` — `DEFAULT_FOREGROUND_OUTPUT_LIMIT_BYTES = 10 * 1024 * 1024` exported. Good — gives downstream tooling a stable name. Confirm the actual default wiring (not in the abbreviated diff): the `ShellExecutionConfig.maxOutputBytes` field is declared at line 105 as optional, so somewhere a `??` fallback to this constant must exist. Reviewer should grep for `maxOutputBytes ??` to confirm the default actually lands.
- `packages/core/src/services/shellExecutionService.ts:391-397` (`getOutputLimitExceededMessage`) — formats limit as `${MiB}MiB` for >=1 MiB, raw bytes otherwise. Test at `shellExecutionService.test.ts:401` asserts `"Command output exceeded the 10 bytes limit"` for `maxOutputBytes: 10`, so the small-value path is exercised. The MiB path uses `.toFixed(1)` — `10485760` renders as `10.0MiB`, which reads slightly off; consider stripping `.0` or using `Math.round` for whole MiB values. Cosmetic.
- `packages/core/src/services/shellExecutionService.ts:120-123` — adds `outputLimitDisabled?: boolean` to the `ActivePty` struct. Used by the "background after start" path to disable the guard mid-stream. The state mutation lives in `ShellExecutionService.background(...)` (not visible in this slice). Verify the same flag exists for `ActiveChildProcess` (line 130 shows it does). Symmetry preserved.
- `packages/core/src/services/shellExecutionService.test.ts:390-411` (PTY guard) — drives 11 bytes through with `maxOutputBytes: 10`, then asserts `result.outputLimitExceeded === true`, the warning copy, and `process.kill(-pid, 'SIGTERM')` (negative pid → process group). Process-group kill is the right call; lone-pid kill leaves grandchildren orphaned for shell pipelines.
- `packages/core/src/services/shellExecutionService.test.ts:413-432` (PTY background bypass) — calls `ShellExecutionService.background(12345, 'default', ...)` *before* emitting the over-limit data, then asserts `result.outputLimitExceeded` is falsy and `mockProcessKill` was *not* called. This locks the load-bearing UX contract: a user/agent who explicitly backgrounds a stream should not be ambushed by the guard.
- `packages/core/src/services/shellExecutionService.test.ts:1452-1490` (child_process parallel) — mirrors both PTY tests. `expect(result.aborted).toBe(false)` at line 1466 is the right invariant — output-limit termination is not the same as user abort and the model needs to distinguish them.
- `packages/core/src/services/executionLifecycleService.ts:27` — adds `outputLimitExceeded?: boolean` to `ExecutionResult`. Optional with no default — downstream consumers reading `result.outputLimitExceeded === true` are safe; consumers using `if (result.outputLimitExceeded)` need a `?? false` only if TS strict-undefined complains. Existing tests cover both shapes.
- `packages/core/src/tools/shell.ts` (+22/-1) and `tools/shell.test.ts` (+18) — not in the abbreviated diff, but the tool layer presumably surfaces the warning in the model-visible output. Reviewer should confirm the `[GEMINI_CLI_WARNING:...]` envelope is preserved verbatim through any post-processing (truncation, ANSI stripping); model-side `read_background_output` redirection only works if it sees the literal string.
- `docs/tools/shell.md:32-37` — user-facing doc update is direct and complete: names the threshold, names the escape hatch (`is_background: true` + `read_background_output`), cites concrete examples (`docker compose logs -f`, `tail -f`, dev servers). No nits.
- Edge case not visible in tests: what happens if `maxOutputBytes` is `0` or negative? Likely either disables the guard or kills immediately. Worth a one-line guard at config-validation time, or a test asserting which behavior wins.

## Verdict

`merge-after-nits` — solid, well-tested guard for a real foreground-context-explosion problem, with symmetric coverage across PTY and child_process and the right SIGTERM-on-process-group semantic. Land after: (1) confirm a `?? DEFAULT_FOREGROUND_OUTPUT_LIMIT_BYTES` fallback wiring exists somewhere, (2) decide on `0`/negative `maxOutputBytes` semantics, (3) optional cosmetic on the `10.0MiB` rendering.
