# charmbracelet/crush PR #2703 — fix re-authorization flow not triggering on certain conditions

Link: https://github.com/charmbracelet/crush/pull/2703
State: MERGED · Author: andreynering · +3 / −1 · Files: 1
(`internal/agent/coordinator.go`)

## Summary
When an OAuth token has expired and the proactive refresh fails, the coordinator previously
returned the refresh error immediately, which short-circuited the request flow. The reactive
re-auth dialog is wired into the *401 response path*, so a hard error before the request was
sent meant users would never see the re-auth prompt — they'd just get an opaque error. This PR
demotes the proactive refresh failure to a `slog.Error` and lets the request proceed with the
stale token, so the server's 401 can drive the existing dialog.

## Strengths
- Diagnoses the real problem precisely: two refresh paths, but only one of them is wired to
  the user-visible recovery UI. Fix preserves a single source of truth for re-auth UX.
- Three-line change with an explanatory comment that names the author and explains the
  intent — future maintainers won't "clean this up" by re-adding the early return.
- Logs the underlying refresh error so operators can still tell *why* the proactive refresh
  failed (network, revoked token, clock skew) even though the user only sees the reactive
  prompt.

## Concerns / Risks
- **Silent fallback for non-401 failure modes.** If the upstream provider returns 403 (token
  revoked rather than expired), 500 (provider outage), or a transport-layer error, the reactive
  dialog never fires and the user sees the raw error. The PR title implies a general fix but
  the actual contract is "the provider must answer with 401 to a stale token". For providers
  that respond with 403 on revoked tokens, the user is now in a worse spot than before — the
  proactive refresh would have surfaced the revocation, but the new path swallows it and then
  shows a generic 403.
- **No metric / counter** on proactive-refresh failure rate. A spike in refresh failures
  (provider OAuth endpoint misbehaving, clock drift, expired refresh token) is now invisible
  in dashboards beyond the log line. Worth a `slog.Warn` plus a counter so this regression
  surface stays observable.
- **Race window**: between the failed proactive refresh and the request hitting the wire, no
  other goroutine is told the token state is suspect. If a parallel coordinator run for the
  same `providerCfg` succeeds in refreshing concurrently, you may end up with one request
  using the stale token and another using the fresh one, briefly. Not a bug per se, but a
  surprise for code reading `providerCfg.OAuthToken` between the two events.
- **No test.** A table test with a mock that fails refresh and asserts the request still goes
  out (and that the eventual 401 triggers the existing dialog hook) would lock this behavior
  in. Right now nothing prevents a future refactor from re-introducing the early return.

## Suggested follow-ups
1. Add a regression test that fakes a refresh failure and asserts the request is still sent.
2. Expose a counter / metric for proactive-refresh failures; the log alone is too easy to
   miss in production.
3. Decide explicitly how 403 / revoked-token responses should funnel into the same re-auth
   dialog, since the PR's premise is that 401 is the canonical re-auth signal.
