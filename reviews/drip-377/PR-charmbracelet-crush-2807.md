# charmbracelet/crush #2807 — fix(summarize): reauthenticate oauth tokens in summarize path

- Head SHA: `b796f550716a2d307f6dd725351c31c10f2d14b9`
- Diff: +38 / -1 across 2 files

## Findings

1. Correctly diagnoses and fixes a real UX bug — when the user triggers `/summarize` after the OAuth token has expired (common with hyper or other OAuth providers after time away), the previous code at `coordinator.go:961-963` returned the 401 unauthorized error directly without refreshing the token, leaving the in-flight summary block spinning forever in the UI with no error rendered. The fix mirrors the existing token-refresh pattern from `Run()` (referenced via `c.refreshOAuth2Token` and `c.refreshApiKeyTemplate` at `:982,989` — both pre-existing helpers).
2. Two-tier refresh strategy at `coordinator.go:966-995` is well-shaped: (a) **proactive** at `:966-971` — if `providerCfg.OAuthToken.IsExpired()` is true *before* the call, refresh-then-summarize (graceful path that avoids the 401 round-trip); (b) **reactive** at `:980-994` — if the call still returns `c.isUnauthorized(err)`, retry-once after refreshing token OR API-key-template (covers the case where the local `IsExpired()` check disagreed with the server, e.g. clock skew or revoked token). The retry-once pattern correctly returns the *original* `err` if the refresh itself fails (`:984-986`) rather than masking the user-visible cause.
3. The companion change at `agent/agent.go:706-711` is the actual UI-spin fix — when `Summarize` returns an error, the previous behavior either deleted the summary message (success path) or just returned the err (failure path), but never marked the in-flight `summaryMessage` as terminal. The new branch at `:706-710` calls `summaryMessage.AddFinish(message.FinishReasonError, "Summarization Error", err.Error())` followed by `messages.Update(ctx, summaryMessage)` so the UI's spinner-loop sees a finished-with-error state and stops spinning. The error string is surfaced in the message body for user diagnosis.
4. Concern (minor): the proactive-refresh failure at `:968-970` only logs an error and proceeds with the (presumably still-stale) token, betting that the reactive-retry path at `:980-994` will catch and re-refresh. This is correct under current semantics but creates an extra failed network round-trip on the proactive-refresh-failed path that didn't previously happen. If the underlying refresh is failing for a non-transient reason (e.g. invalidated refresh token), the user now sees two log lines and one wasted API call before the eventual error — minor cost, but worth a comment at `:970` clarifying the intentional fall-through.

## Verdict

**Verdict:** merge-as-is
