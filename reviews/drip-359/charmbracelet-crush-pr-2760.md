# charmbracelet/crush PR #2760 — feat(bedrock): add aws_auth_refresh option for AWS credential refresh

- Repo: `charmbracelet/crush`
- PR: #2760
- Head SHA: `1bd7ba6dbe7f761d5f8f8ab97c4ee0fa22a480ac`
- Author: `carlosgrillet`
- Updated: 2026-04-30T09:57:06Z
- Verdict: **merge-after-nits**

## What it does

Adds a new `Options.AWSAuthRefresh string` config knob (`internal/config/config.go:260`) that names a shell command crush runs (a) at startup *before* configuring the Bedrock provider in `internal/config/load.go:289-291`, and (b) at request time *after* a 401/403 from Bedrock, before retrying the failed turn (`internal/agent/coordinator.go:233-242`).

Three layers:

1. **`internal/config/load.go:752-768`** — new `RunAWSAuthRefresh(cmd string)` runs `sh -c <cmd>` with `Stdin/Stdout/Stderr` attached to the current terminal so interactive SSO/MFA prompts can complete. Empty/whitespace-only command is a documented no-op.
2. **`internal/agent/coordinator.go:233-242`** — after a request failure, if the provider is Bedrock and `AWSAuthRefresh != ""` and the error is `isBedrockAuthError` (HTTP 401 or 403), the coordinator runs the refresh command and retries the turn once via `return run()`. On refresh failure it returns the original Bedrock error.
3. **`internal/agent/coordinator.go:968-977`** — `isBedrockAuthError` matches `*fantasy.ProviderError` with `StatusCode` in `{401, 403}`. Comment correctly notes "SigV4 expired-session tokens surface as 403 (ExpiredTokenException), while a missing signer or disabled key is 401."

Tests:

- `internal/config/aws_auth_refresh_test.go:1-29` — covers empty-string no-op, whitespace-only no-op, `true` succeeds, `false` errors, and a `test 1 = 1 && exit 0` shell-expansion case.
- `internal/agent/bedrock_auth_test.go:1-32` — table test pinning the 401/403-true vs 500/429/nil/non-provider-false matrix for `isBedrockAuthError`.

## Strengths

- **Real operator pain point.** Bedrock SSO sessions and MFA-derived STS credentials expire on a schedule that is independent of the crush process lifetime; the previous behavior was "the user gets a 401, has to quit, run their `aws sso login` shell, and restart crush." The two-phase wiring (startup + on-401/403 retry) covers both the cold-start and mid-session expiry paths.
- **Correct status-code set.** AWS Bedrock's SigV4 path returns 403 with `ExpiredTokenException` when the session token is past its expiry, and 401 when no signer is configured. The comment at `coordinator.go:966-968` captures this distinction. 429 (rate-limit) and 500 (server) are correctly excluded — refreshing creds on those would mask real upstream issues.
- **Safe execution model.** `sh -c <cmd>` plus inherited tty handles is the right primitive: it lets `aws sso login`, `duo-sso`, `gimme-aws-creds --mfa`, and similar tools prompt the user. The empty-string no-op means existing crush configs don't change behavior.
- **Defensive on refresh failure.** If `RunAWSAuthRefresh` returns an error, the coordinator returns the **original** Bedrock error (`return nil, originalErr` at line 240) — so the user sees "your Bedrock call failed with 403" rather than a confusing "your refresh script failed" that obscures the real problem.
- **Tests pin the contract.** `bedrock_auth_test.go` tests both the positive cases (401, 403) and negative cases that easily regress (500, 429, nil, non-provider error). `aws_auth_refresh_test.go` covers the trim/empty/exit-code matrix.

## Nits / concerns

1. **One-shot retry is reasonable but undocumented.** `coordinator.go:241` calls `return run()` exactly once after the refresh; if the refresh "succeeds" (exit 0) but the new credentials are still bad (e.g. the SSO script silently used a stale cache), the user gets a second 401/403 with no further retry. This is the right default — infinite retries on auth would be much worse — but the JSON schema description on `AWSAuthRefresh` (line 260) should mention "runs at most once per request after a 401/403."

2. **No timeout on the refresh command.** `c.Run()` at `load.go:767` will block indefinitely if the SSO browser flow never completes. Acceptable for an interactive-tty operator, dangerous for headless/CI invocations of crush. Consider a `context.WithTimeout` (configurable, default e.g. 5 min) and surface a clear "aws_auth_refresh exceeded timeout" error.

3. **`exec.Command("sh", ...)` is POSIX-only.** Windows users with `crush` configured against Bedrock won't get the refresh path — `sh` won't resolve. Either document this in the schema description as POSIX-only, or branch on `runtime.GOOS == "windows"` and shell out via `cmd /c` (with the caveat that interactive auth flows on Windows usually go through Edge/IE handlers anyway and may need a different mechanism).

4. **Startup invocation is unconditional on `AWSAuthRefresh != ""`.** `load.go:289-291` calls `RunAWSAuthRefresh(c.Options.AWSAuthRefresh)` *before* the existing `hasAWSCredentials(env)` check at line 293. If the user has valid env-var credentials already, the refresh script still runs, which for many `aws sso login --no-cache=false` style scripts is a no-op but for an `exec duo-sso ... -valid-session-threshold 7200` style command (the example in the schema) it triggers an interactive prompt the user didn't need. Consider running the refresh *only* when `!hasAWSCredentials(env)`, OR when the script is documented as idempotent / self-throttling — and make the example in the schema description match that contract.

5. **`shell expansion works` test asserts `&& exit 0` works** (line 26 of `aws_auth_refresh_test.go`). Confirms `sh -c` mode but doesn't test what happens with shell metacharacters embedded in user input — not a security concern since the operator owns the config file, but worth a comment that `aws_auth_refresh` is implicitly a shell-injection sink and therefore should not be sourced from any untrusted location.

6. **Coordinator retry vs. existing OAuth-401 path.** `coordinator.go` already has `refreshOAuth2Token` (line 968-on, partially shown) which handles a similar 401-then-refresh-then-retry pattern for OAuth2 providers. The new Bedrock branch is a parallel implementation; consider whether they can share a small `retryAfterCredentialRefresh(provider, refreshFn, isAuthErrFn)` helper to keep the patterns in sync as either one evolves.

## Verdict
**merge-after-nits** — well-motivated feature that fills a real Bedrock operator gap, the status-code set is correct, the tty-inheriting shell exec is the right primitive, and the test coverage is appropriate. Add a configurable timeout, document the POSIX-only nature, gate the startup invocation on missing creds, and consider unifying the retry path with the existing OAuth refresh flow before landing.
