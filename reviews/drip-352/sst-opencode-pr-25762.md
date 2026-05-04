# sst/opencode #25762 — fix: prevent shell commands from killing all Node.js processes

- URL: https://github.com/sst/opencode/pull/25762
- Head SHA: `4c7cf5639030b394337f21ded6afecea7c84ce3d`
- Author: Xelson431
- Size: +29 / -0 across 3 files

## Comments

1. `packages/opencode/src/tool/shell.ts:31-43` — `DANGEROUS_COMMAND_PATTERNS` regexes are reasonable but overlap: pattern 1 (`taskkill .*\/F .*\/IM node\.?exe`) is fully subsumed by pattern 2 (`taskkill .*\/IM node\.?exe`). Drop the first to keep the list minimal.
2. `packages/opencode/src/tool/shell.ts:33` — `/Get-Process\s+.*node\s*\|\s*Stop-Process/i` won't catch the equally common `gps node | kill` (alias) or `Stop-Process -Name node`. Consider adding `/Stop-Process\s+(-Name\s+)?["']?node/i`.
3. `packages/opencode/src/tool/shell.ts:616-619` — The check runs *after* `params.timeout` validation but *before* `Shell.ps`. Good placement, but please surface the block as a structured tool error (set `metadata.blocked = true`) rather than throwing, so the model gets a clean signal instead of a stack trace.
4. `packages/opencode/src/tool/shell.ts:31-35` — A determined model can defeat regex with `node\x65xe`, `n""ode.exe`, or by chaining via `bash -c`. Worth a comment that this is a guardrail against accidental self-termination, not a security boundary.
5. `packages/opencode/src/session/system.ts:61-66` and `src/tool/shell/shell.txt:11-16` — The same prose appears in both places. Extract into a single constant or template so they can't drift.
6. Missing test coverage. Add a small unit test that calls the tool with `killall node` and asserts the structured-error path; otherwise regressions will slip through.

## Verdict

`merge-after-nits`

## Reasoning

The change addresses a real and recurring failure mode (the model self-terminating via `taskkill /F /IM node.exe` on Windows), the implementation is small and self-contained, and the prompt update reinforces the runtime guard. The nits are: dedupe the regex list, broaden the PowerShell match, route the rejection through structured tool metadata, and add at least one test. None of these block merge in spirit, but landing without a test sets up the next contributor to silently weaken the guard.
