# charmbracelet/crush#2807 — fix(summarize): reauthenticate oauth tokens in summarize path

- PR: https://github.com/charmbracelet/crush/pull/2807
- Head SHA: `b796f550716a2d307f6dd725351c31c10f2d14b9`
- Size: +38/-1
- Verdict: **merge-after-nits** (man)

## Shape

Closes a real reliability gap where invoking `Summarize` on a long-running
session whose OAuth token had expired since session start surfaced a 401 to
the user instead of transparently refreshing — `Run()` already had the
proactive-refresh-then-retry-on-401 dance, but `Summarize()` was a thin
pass-through that didn't.

Two-file change:

1. `internal/agent/agent.go:706-712` — when summarization fails, before
   returning the error, mark the in-flight `summaryMessage` as
   `FinishReasonError` with the actual error string and call
   `messages.Update(ctx, summaryMessage)` so the UI's "summarizing…" spinner
   stops. Without this, an OAuth-401-failed summarize left the message in
   pending state and the spinner ran forever (PR is presumably motivated by
   that exact bug report).
2. `internal/agent/coordinator.go:964-995` — `Summarize` now mirrors the
   `Run()` token-refresh shape:
   - **Proactive**: if `providerCfg.OAuthToken != nil &&
     providerCfg.OAuthToken.IsExpired()`, call `refreshOAuth2Token` *before*
     the first attempt and log on failure but proceed (sensible — try the
     stale token, the server may still accept it).
   - **Reactive**: if the first attempt errors with `c.isUnauthorized(err)`,
     branch on `OAuthToken != nil` (refresh and retry) vs.
     `strings.Contains(providerCfg.APIKeyTemplate, "$")` (refresh API key
     template and retry).
   - The retry path is a closure `summarize := func() error { ... }` so the
     two call sites stay byte-identical and don't drift.

## Notable observations

- The closure pattern at `coordinator.go:973-975` is the right shape and
  matches what `Run()` does — keeps "the actual call" defined exactly once
  even though it's invoked twice.
- `slog.Error("Failed to refresh OAuth2 token before summarize. Proceeding
  with existing token.", ...)` at `:970` is the correct log level — it's a
  warning condition that doesn't yet block, and the next attempt may still
  succeed if the token has a slack window. But could also be `slog.Warn` —
  `slog.Error` reads as "we already failed", which we haven't.
- `agent.go:706-712`: `AddFinish(message.FinishReasonError, "Summarization
  Error", err.Error())` then `Update`. If `Update` itself errors, we return
  `updateErr` and the *original* summarization error is lost. Consider
  `errors.Join(err, updateErr)` so both surface.

## Concerns / nits

- No test in the diff. The two contracts to pin: (a) expired token →
  proactive refresh → success (no retry needed), and (b) valid-looking token
  → 401 → reactive refresh → retry succeeds. Both are mockable against a
  fake provider config.
- The reactive-401 branch only handles `OAuthToken != nil` and `APIKeyTemplate
  contains "$"`. If neither is true (static API key), the retry is silently
  skipped — verify `Run()` has the same shape so behavior stays symmetric.
- Single-shot retry only — if the refreshed token is *also* rejected (race
  with provider-side revocation), the second 401 propagates. That's probably
  fine for summarize (don't retry forever), but worth a one-line comment.
- The "Summarization Error" string at `agent.go:709` is hardcoded English;
  if Crush ships any i18n, this will need to flow through the translation
  layer.
