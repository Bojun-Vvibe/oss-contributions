# google-gemini/gemini-cli #26256 — fix(shell): stop foreground commands after excessive output

- Head SHA: `886b0afe8e74b11af6c65a26e1d3b82bc8f4db37`
- Files: `docs/tools/shell.md`, `packages/core/src/services/executionLifecycleService.ts`, `packages/core/src/services/shellExecutionService.{ts,test.ts}`, `packages/core/src/tools/shell.{ts,test.ts}`
- Size: +226 / -4

## What it does

Foreground shell commands invoked via the `run_shell_command` tool
could previously stream unbounded output into the model's context
window — `docker compose logs -f`, `tail -f`, dev servers, etc.
The PR adds a configurable per-command output cap (default 10 MiB,
constant `DEFAULT_FOREGROUND_OUTPUT_LIMIT_BYTES` at
`packages/core/src/services/shellExecutionService.ts:50`) that, when
exceeded, kills the process group and returns a structured "this was
stopped, run it in the background instead" message.

Plumbing:

- `ExecutionResult` (`executionLifecycleService.ts:27`) gains a new
  optional `outputLimitExceeded?: boolean` field.
- `ShellExecutionConfig` (`shellExecutionService.ts:105`) gains
  `maxOutputBytes?: number`.
- Two parallel implementations get the same enforcement, one for
  the child-process path (~line 558) and one for the PTY path
  (~line 1050). Each defines a `maybeStopForOutputLimit(bytesReceived)`
  closure that:
  1. Early-returns if `maxOutputBytes` is unset, the limit was
     already tripped, the process exited, the limit is disabled
     (background flag set), or there's no PID.
  2. Increments `streamedBytes += bytesReceived`.
  3. If `streamedBytes > maxOutputBytes`, sets
     `outputLimitExceeded = true` and calls
     `killProcessGroup({ pid, escalate: true, isExited: () => exited })`
     (PTY variant also passes `pty: ptyProcess` for proper
     teardown). Failures from `killProcessGroup` are debug-logged
     and swallowed.
- Each `handleOutput` callback calls `maybeStopForOutputLimit(data.length)`
  before doing anything else, so the trip happens on the *receiving
  byte count* rather than the post-decode string length —
  important because `data.length` is the raw byte cost and it's
  what decides whether the model's context can hold it.
- On exit, both code paths append a static warning to the captured
  output if the limit was tripped (via `getOutputLimitExceededMessage`
  at `shellExecutionService.ts:391-397`) which renders MiB for
  `>= 1 MiB` thresholds and bytes otherwise.
- `ShellExecutionService.background(...)` at `:1473-1479` flips
  `activePty.outputLimitDisabled = true` and
  `activeChild.state.outputLimitDisabled = true` so a foreground
  command that gets promoted to background mid-stream has its limit
  immediately disabled.
- The tool wrapper at `packages/core/src/tools/shell.ts:600-605`
  passes `maxOutputBytes` only when `is_background` is false — so
  background invocations from the tool surface *start* without the
  limit, and foreground invocations get the configured value or the
  10 MiB default.
- The model-facing message at `shell.ts:691-700` distinguishes
  `outputLimitExceeded` from `aborted` and explicitly tells the
  model to retry with `is_background=true` and inspect with
  `read_background_output` — which is the actionable next step.

Tests cover four scenarios in both code paths (PTY and
child-process):
- `should stop a foreground PTY when output exceeds the configured limit`
  at `shellExecutionService.test.ts:390-410` — exceed the limit,
  assert `outputLimitExceeded === true`, assert the warning
  string is in `result.output`, assert `process.kill` was called
  with the negative PID (process-group kill) and `SIGTERM`.
- `should disable the PTY output limit after moving to background`
  at `:412-431` — call `.background(...)` mid-stream, then
  exceed the would-be-limit, assert `outputLimitExceeded` is
  falsy and `process.kill` was *not* called.
- Same two for the child-process fallback at `:1456-1493`.
- A tool-level test at `tools/shell.test.ts:1007-1023` asserts the
  model-facing `llmContent` contains "produced too much output",
  "is_background=true", and "read_background_output".

## What works

Putting the trip-check on `data.length` (raw bytes) rather than
post-decode character count is correct — the limit exists to bound
context-window cost, and byte count is the right currency
(multi-byte UTF-8 like Chinese log lines would otherwise undercount
by ~3x).

The "background flag disables the limit" hook at
`shellExecutionService.ts:1473-1479` is the right call: a user who
explicitly asks for `is_background: true` (or who gets bumped to
background mid-run) is signalling "I'll consume this on demand,
don't bound it" and the bounded snapshot inspection happens
through `read_background_output` instead. The flag is checked
inside `maybeStopForOutputLimit` so a race where data arrives
between the `background()` call and the next `handleOutput`
correctly sees the disabled flag.

Process-group kill via negative PID (`-mockPtyProcess.pid`) with
SIGTERM is the right escalation for shell wrappers that fork
children — `docker compose logs -f` spawns a long-lived child
that won't die from SIGTERM to the parent alone. The
`escalate: true` option on `killProcessGroup` (per the test
asserting `mockProcessKill` is called) implies an eventual
SIGKILL fallback for processes that ignore SIGTERM.

The model-facing message at `shell.ts:691-700` does the right thing
by being prescriptive (`run with is_background=true and inspect with
read_background_output`) rather than just describing the failure.
Models recover from prescriptive errors much better than from
descriptive ones.

The 10 MiB default is well-chosen: large enough that any normal
build / test / one-shot command fits, small enough that a
runaway log stream gets caught in seconds rather than minutes.

## Concerns

1. **The exit-path message-append has a TOCTOU.** At
   `shellExecutionService.ts:747-751` (child path) the warning is
   appended to `combinedOutput` only if `outputLimitExceeded` is
   true at exit time — but the check on the limit happens in
   `maybeStopForOutputLimit` *before* `data` is appended to
   `state.output`, so under heavy streaming the captured output
   may end well past the `maxOutputBytes` mark (because additional
   chunks land between trip-time and SIGTERM-takes-effect). The
   warning correctly says "exceeded the 10 MiB limit" but the
   `output` field can be larger than 10 MiB. A second guard that
   truncates `state.output` to `maxOutputBytes` would tighten the
   contract.
2. **Background-promotion race window.** If `background()` is
   called from a different async context than the `handleOutput`
   callback runs in, there's a small window where a chunk is
   already mid-flight through `maybeStopForOutputLimit` when
   `outputLimitDisabled` flips true. The early-return checks
   `ShellExecutionService.activePtys.get(ptyPid)?.outputLimitDisabled`
   on every call so the *next* chunk after the flip sees the new
   value, but the in-flight chunk that already incremented
   `streamedBytes` past the threshold will still trip. The
   second test (`should disable the PTY output limit after moving
   to background`) calls `background()` *before* the
   over-limit chunk so it doesn't exercise this race. Probably
   fine in practice, but a comment noting "in-flight chunks may
   still trip the limit even after .background() is called" would
   set expectations.
3. **No test for the limit being explicitly set to zero.** A
   user who configures `maxOutputBytes: 0` would see every
   foreground command killed on its first byte, which is silly
   but probably the literal correct interpretation. A guard
   that treats `<= 0` as "disabled" (matching the
   `undefined`-is-disabled semantics) would be more forgiving.
4. **Two parallel implementations of `maybeStopForOutputLimit`.**
   The closure exists once for the child-process path
   (`:632-660`) and once for the PTY path (`:1056-1086`) with
   nearly identical logic but different state-storage shapes
   (`state.outputLimitDisabled` vs
   `activePty.outputLimitDisabled`). Easy to drift later.
   Extracting a `class OutputLimitTracker` with a single
   implementation would unify them.
5. **`docs/tools/shell.md` mentions only the 10 MiB default** but
   not the configurability via `maxOutputBytes`. If
   `shellExecutionConfig.maxOutputBytes` is user-facing, the
   doc should mention it; if not, the constant should be
   `private` rather than `export const`.

Verdict: merge-after-nits

## What I learned

The right shape for a streaming-output cap is "trip on receiving
byte count, don't truncate the output buffer" — let the
process exit at its own pace after SIGTERM, capture whatever
arrives in the meantime, and append a structured marker. The
alternative (truncate-as-you-go) creates a worse failure mode
where the model sees a clean-looking but artificially-cut
output and may not realize the command was stopped. The
"prescriptive recovery message" pattern at `shell.ts:691-700`
is also worth borrowing: every tool-error message should end
with the literal next action the model should take, not just
a description of what went wrong.
