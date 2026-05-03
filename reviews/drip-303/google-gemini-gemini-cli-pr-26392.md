# google-gemini/gemini-cli #26392 — fix(windows): Resolve hangs, zombie processes, and improve subagent reliability

- URL: https://github.com/google-gemini/gemini-cli/pull/26392
- Head SHA: `4ff5a60f159b0cc6b3a1e1bc1edc304169b8d27e`
- Scope: +170 / -49 across 9 files
- Claims to fix: #26393

## Summary

PR description bundles **six unrelated changes** under a single "Windows
fixes + subagent reliability" header:

1. Windows startup hang from slow `Get-CimInstance` WMI scan → 5s timeout
2. Zombie processes on cancel → use `taskkill /F /T` on Windows
3. Slash command recognition broken by trailing newlines → `.trim()`
4. Memory bloat from in-memory log accumulation → externalize to
   `~/.gemini/logs/latest.log` with 500-line rolling buffer
5. Infinite subagent error loops → return explicit error objects + 3-failure
   circuit breaker
6. `WriteFile` silently skipping content when edit-corrector returns empty
   → fall back to original proposed content

## What I checked

- `packages/cli/src/commands/gemma/setup.ts` (+49/-32) — adds an
  `AbortController` with 30s timeout around the `fetch()` download. **This
  is change #7, not in the PR description list** — the file is `gemma/setup.ts`
  (model download timeout), nothing to do with the WMI scan fix mentioned
  as item #1. Mismatch between description and diff.
- `packages/cli/src/ui/components/LoadingIndicator.tsx:71` — single-line
  change adding `(${elapsedTime}s)` to the "Thinking..." label. **Also not
  in the PR description.**
- `packages/cli/src/ui/utils/commandUtils.ts:42+` — `isSlashCommand`
  trim fix. This matches description item #3.
- `packages/cli/src/utils/activityLogger.ts` (+33/-3) — log rotation /
  externalization. Matches item #4.
- `packages/core/src/agents/local-executor.ts` (+23) and
  `local-invocation.ts` (+5/-2) — subagent error propagation. Matches
  item #5.
- `packages/core/src/ide/process-utils.ts` (+2) — only 2 lines added; the
  PR description's "5s timeout for PowerShell + empty map fallback" claim
  doesn't fit a 2-line diff. **Suspicious.**
- `packages/core/src/services/executionLifecycleService.ts` (+34/-1) —
  presumably the `taskkill` integration. Matches item #2.
- `packages/core/src/tools/write-file.ts` (+19/-7) — fallback for
  edit-corrector. Matches item #6.

## Concerns — major

1. **Description ≠ diff.** Item #1 (WMI scan timeout) cannot be a 2-line
   change to `process-utils.ts`. Either the description is wrong, the
   diff is incomplete, or the fix is actually elsewhere and not labeled.
   Reviewer cannot validate item #1 as written.
2. **Two undocumented changes.** `gemma/setup.ts` (download timeout) and
   `LoadingIndicator.tsx` (elapsed-time label) are real changes that
   aren't called out anywhere. Both look reasonable individually but
   bundling them silently into a "Windows fixes" PR is bad hygiene.
3. **Six fixes in one PR.** Even ignoring the two extras, six independent
   bugs in one PR is far too many for safe review. Each has its own
   risk profile, regression surface, and rollback story:
   - The `taskkill /F /T` change can leak descendant processes if the
     PID has been recycled. Needs careful Windows-side review.
   - The 500-line rolling log buffer is a behavior change that loses
     historical log data — needs a separate decision.
   - The 3-failure circuit breaker for subagents has product implications
     (when does the agent give up?).
   - The WriteFile fallback silently changes which content gets written
     when the corrector misbehaves — that's a security-adjacent change.
4. **No test files in the diff.** Six bug fixes, zero new tests. PR
   description says "Confirmed startup time is now <2s" / "Verified via
   Task Manager" — all manual, none reproducible in CI.
5. **Item #6 is potentially destructive.** "WriteFile occasionally reports
   success without writing content" → "fall back to original proposed
   content if the AI correction phase fails". If the LLM proposed a
   destructive write and the corrector was supposed to mitigate it, the
   fallback now silently bypasses the corrector. Threat model unclear.

## Nits

- The HTML-encoded characters in the PR body (`\\Get-CimInstance\\`,
  `\\process.kill(-pid)\\`, `\\\"What happened, Why...\\\"`) suggest the
  description was machine-generated or pasted through a transform that
  mangled it. Worth cleaning up before merge.

## Risk

High. Six unrelated changes bundled, two undocumented, no test coverage,
manual-only validation, and at least one (WriteFile fallback) with
non-obvious safety implications. Hard to review safely as a single unit.

## Verdict

`request-changes` — split into one PR per fix. Each individual fix may
well be sound (the slash-command trim and log externalization look
reasonable on their face), but the current bundle is unreviewable. At a
minimum the description must match the diff before any merge consideration.
