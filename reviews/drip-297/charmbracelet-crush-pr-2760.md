# Review: charmbracelet/crush PR #2760

- **Title:** feat(bedrock): add aws_auth_refresh option for AWS credential refresh
- **Author:** carlosgrillet (Carlos Grillet)
- **Head SHA:** `1bd7ba6dbe7f761d5f8f8ab97c4ee0fa22a480ac`
- **Verdict:** merge-after-nits

## Summary

Adds top-level `options.aws_auth_refresh` string that runs a shell
command to refresh AWS credentials before Bedrock calls. Mirrors a
similar setting in another mainstream agent CLI. Runs once on startup
(inside `configureProviders`) and on retry when Bedrock returns 401 or
403 (the latter covers `ExpiredTokenException`). 106 added across 5
files, with unit tests for both the runner and the auth-error
classifier.

## Specific-line comments

- `internal/config/load.go:752-766` (`RunAWSAuthRefresh`) — `exec.Command("sh", "-c", cmd)`
  with `Stdin/Stdout/Stderr` attached to the current terminal is
  exactly right for SSO/MFA flows. Empty/whitespace-only command is a
  no-op, which matches the documented "set this to disable" pattern.
- `internal/config/load.go:289-291` — startup-side call inside the
  `case catwalk.InferenceProviderBedrock:` arm. Failing config load on
  non-zero exit is the right default — surfacing the failure early
  beats a cryptic SigV4 error 30 seconds in.
- `internal/agent/coordinator.go:233-243` — retry branch only runs when
  `providerCfg.Type == bedrock.Name`, the option is set, and the error
  matches `isBedrockAuthError`. Correctly returns `originalErr` (not a
  fresh wrapper) if the refresh command itself fails, so the user sees
  the *real* Bedrock error, not a confusing "refresh failed" stack.
- `internal/agent/coordinator.go:968-978` (`isBedrockAuthError`) —
  401 OR 403 is correct for AWS. The PR comment explicitly calls out
  `ExpiredTokenException` → 403, which is the load-bearing fact. Good
  decision to treat 403 as auth-retryable *only* on the Bedrock path
  (other providers may use 403 for permission denials that are NOT
  fixable by re-running an auth command).
- `internal/config/aws_auth_refresh_test.go:21-28` — tests cover empty,
  success, failure, and shell expansion cases. Good. Missing case:
  command that hangs forever — there's no timeout. See risks.

## Risks / nits

- **No timeout on the refresh command.** `exec.Command(...).Run()` will
  block indefinitely if the SSO browser flow hangs, and
  `coordinator.Run` will sit on it on every retry. Consider wrapping
  with `exec.CommandContext(ctx, ...)` and respecting the agent
  context's cancellation, or at minimum a generous default timeout
  (e.g. 5 minutes) configurable via the same `options` block.
- **Single-retry, no backoff.** If the refresh succeeds but the new
  creds still fail (e.g. role assumption returns wrong principal), the
  retry just gives the same auth error. That's acceptable but worth
  noting in the docs — users will sometimes need to re-run manually.
- **Non-deterministic test:** `test "shell expansion works"` runs
  `test 1 = 1 && exit 0`. Works on POSIX `sh`, but on a misconfigured
  CI worker without `/bin/sh` (rare, but: scratch containers) this
  fails. Use a Go-internal command if portability matters.
- The PR mentions a sibling `awsCredentialExport` (parsing env vars
  from stdout) is intentionally out of scope. Worth a TODO comment in
  the code or a follow-up issue link in the docs so it doesn't get
  lost.

## Verdict justification

Tight, well-tested, and the design choices (401|403, idempotent
command expectation, fail-fast on startup, fail-soft on retry) are
right. The missing timeout is the only real concern — easy to address
in a follow-up. **merge-after-nits.**
