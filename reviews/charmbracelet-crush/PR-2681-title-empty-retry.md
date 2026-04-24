# PR-2681 — fix(agent): retry title generation with large model on empty output

[charmbracelet/crush#2681](https://github.com/charmbracelet/crush/pull/2681)

## Context

`internal/agent/agent.go` — `generateTitle` previously capped the
small-model title call at 40 output tokens (`var maxOutputTokens int64
= 40`). For a non-reasoning small model that's plenty for a one-line
title; for a "non-reasoning" model that secretly burns output budget on
hidden reasoning (the PR specifically calls out GPT-5 mini), 40 tokens
get consumed entirely by the hidden reasoning trace and the visible
content comes back empty with `finish_reason=stop`. The session then
gets named whatever the legacy fallback was (typically the default
`DefaultSessionName`).

The fix:

1. Lifts the small-model cap to a named constant
   `smallTitleMaxTokens int64 = 256` (and `fallbackTitleMaxTokens int64 =
   1024` for the large model when its catwalk config doesn't advertise
   a default).
2. Adds `isUsableTitleResponse(resp)` — checks `resp != nil` and
   `strings.TrimSpace(resp.Response.Content.Text()) != ""`.
3. After the small-model call returns successfully, if the response is
   *empty* and the model used wasn't already the large one, retries
   with the large model using `cmp.Or(largeModel.CatwalkCfg.
   DefaultMaxTokens, fallbackTitleMaxTokens)` as the cap. Only swaps
   in the retry result if it's also `isUsableTitleResponse`.
4. Tightens log keys: `"err" → "error"` for consistency.
5. Fixes a stale `// generates a session titled` typo to `title`.

## Why it matters

Empty title responses with `finish_reason=stop` are the failure mode
where everything looks fine to the SDK — no error, no usage warning,
no rate-limit signal — and yet the user sees `Untitled session` in
their sidebar forever. The cost is wasted tokens (40+ for the empty
small call, plus the large fallback) and a bad UX. The category of
"reasoning models silently eat the output budget" is going to be
increasingly common, so the symptom-detection approach
(`isUsableTitleResponse`) is the right shape.

## Strengths

- Picking 256 instead of 40 as the small-model default is correctly
  conservative: still cheap, but enough headroom that most reasoning-
  capable smalls (GPT-5 mini in the PR comment, plus presumably others
  in the same class) emit visible text. Doesn't kick the can.
- The retry only fires if `model.CatwalkCfg.ID != largeModel.CatwalkCfg
  .ID` — i.e. when small and large are configured to the *same* model
  (which happens when only one model is set), the retry is skipped.
  Avoids a pointless duplicate call.
- `isUsableTitleResponse` does the right minimal check: trims
  whitespace before deciding emptiness. A model returning `"\n"`
  shouldn't count as a usable title.
- Both fallback paths converge on `if resp == nil` for the
  default-name escape, so a retry that *also* returns nothing falls
  through cleanly to `a.sessions.Rename(ctx, sessionID,
  DefaultSessionName)`.

## Concerns / risks

- The retry path duplicates the entire `newAgent(largeModel.Model,
  titlePrompt, ...).Stream(ctx, streamCall)` call from the original
  fallback. If the original small-model `err` path is also taken, the
  flow becomes: small fails → large try → if large returns empty, no
  second large retry → fall through. That asymmetry (only the
  small-success-empty path triggers the swap) is correct but hard to
  see at a glance. A small comment would help future readers.
- `cmp.Or(largeModel.CatwalkCfg.DefaultMaxTokens, fallbackTitleMaxTokens)`
  uses `DefaultMaxTokens` as the cap. For a frontier reasoning model
  this could be e.g. 32k — generating a session title with a 32k cap
  is expensive if the model decides to chain-of-thought before
  emitting. There's no upper clamp.
- The log message `"Large model fallback failed after unusable small
  model response"` only fires on a hard `retryErr`. If the retry comes
  back empty *too*, there's a TODO-ish comment "Otherwise both models
  returned empty content; fall through and let the default-title path
  below handle it" but no log line. That's the case operators will
  most want to see in metrics ("how often does our model class fail to
  produce a one-line title?").
- No test coverage in the diff for the new retry branch. The behavior
  is purely empirical — easy to regress.
- `isUsableTitleResponse` only checks for *any* non-whitespace text. A
  model that emits a single `.` or `-` would pass and produce a
  garbage title. Probably acceptable for v1.

## Suggested follow-ups

- Add a unit test using a fake `fantasy.AgentResult`: small returns
  empty → large returns "My Session" → assert title is "My Session";
  small returns empty → large returns empty → assert
  `DefaultSessionName`.
- Add a `slog.Warn` (not silent) for the both-empty terminal case so
  operators can quantify the drop rate.
- Consider clamping the retry budget to e.g. `min(DefaultMaxTokens,
  4096)` — a title shouldn't ever cost a 32k completion.
- Consider extending `isUsableTitleResponse` to require ≥3 non-whitespace
  characters or to reject pure-punctuation responses.
