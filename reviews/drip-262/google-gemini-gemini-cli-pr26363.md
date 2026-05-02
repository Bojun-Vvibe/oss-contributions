---
pr: https://github.com/google-gemini/gemini-cli/pull/26363
head_sha: 171683efcc973e6103c7f189fdca9692fd29d99e
author: Akash504-ai
additions: 24
deletions: 0
files_touched: 2
---

# Scope

One-line bug fix in `validateNonInteractiveAuth`: when output mode is
JSON and authentication fails, the function previously called
`handleError(...)` but did not rethrow, so execution silently continued
and the function returned `undefined` instead of terminating. Adds a
single regression test. The PR title mentions "coreEvents listener
cleanup on all exit paths in nonInteractiveCli" but the actual diff is
strictly the throw-after-handleError fix in
`validateNonInterActiveAuth.ts` — title/body mismatch (see below).

# Specific findings

- `packages/cli/src/validateNonInterActiveAuth.ts:59` (head `171683e`)
  — `throw error;` is added immediately after the `handleError(...)`
  call inside the `OutputFormat.JSON` branch. This matches the TEXT-mode
  branch's behavior, where `runExitCleanup()` is awaited and execution
  terminates. Correct minimal fix.
- One concern: `handleError(...)` is called with
  `ExitCodes.FATAL_AUTHENTICATION_ERROR` and conventionally invokes
  `process.exit(...)` itself in many CLI codebases. If
  `handleError` *does* call `process.exit` here, the new `throw error`
  is dead code at runtime but still useful to satisfy the type system
  and protect against future refactors that defer or skip the exit.
  Worth a one-line comment confirming the intent.
- `packages/cli/src/validateNonInterActiveAuth.test.ts:469-490` — the
  new regression test asserts `threw === true` and
  `result === undefined`. The combination is exactly what's needed to
  pin the bug ("must throw, must not return a value"). Good.
- The PR title ("Fix: ensure coreEvents listener cleanup on all exit
  paths in nonInteractiveCli") describes a *different* fix than what
  the diff actually delivers (the diff is purely about throwing on
  auth failure in JSON mode; nothing about `coreEvents` listener
  cleanup is touched). The PR body's "Summary / Details / Fix"
  sections describe the actual diff correctly, so the title appears
  to be a copy-paste error from another PR. Maintainer should ask for
  a title rename before merge to keep the changelog/commit history
  searchable.
- The PR's "How to Validate" section references the test file path
  with mixed casing
  (`packages/cli/src/validateNonInterActiveAuth.test.ts`) — the actual
  filename in the diff is `validateNonInterActiveAuth.test.ts`, which
  matches; mixed-casing in case-insensitive filesystems is a known
  footgun in this repo's package layout but not introduced by this PR.

# Suggested changes

1. **Blocking-ish:** rename the PR title to accurately describe the
   fix, e.g. `fix(cli): rethrow auth error in JSON mode of
   validateNonInteractiveAuth`. Misleading commit titles bite later
   when bisecting or scanning release notes.
2. Add a brief inline comment at the new `throw error;` line explaining
   that this guards against `handleError` deferring or skipping
   `process.exit`, so callers can rely on exception flow.
3. Optional: also assert in the test that `handleError` was called
   with `ExitCodes.FATAL_AUTHENTICATION_ERROR` to lock the
   error-classification contract.

# Verdict

`merge-after-nits`

# Confidence

High — the code change is one line, the test is appropriate, and the
remaining concerns are about communication (title) and defensive
documentation rather than correctness.

# Time spent

~7 minutes.
