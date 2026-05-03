# charmbracelet/crush PR #2681 — fix(agent): retry title generation with large model on empty output

- **Author:** iuga
- **Head SHA:** `e2caa0ba34b024acf5429a40b818c4550d1e3f87`
- **Base:** `main`
- **Size:** +43 / -5 (1 file)
- **Fixes:** #1189 (sessions saved as "Untitled Session" with reasoning small models)

## What it does

`internal/agent/agent.go`: when the small model used for title generation
returns empty visible text (because it spent its entire output budget on
hidden reasoning), fall back to the large model once instead of accepting
"Untitled Session". Also raises the small-model output cap from 40 → 256
tokens.

## Diff observations

`internal/agent/agent.go:54-65` adds two named constants with documentation
explaining *why* 256 / 1024:

```go
smallTitleMaxTokens   int64 = 256
fallbackTitleMaxTokens int64 = 1024
```

`internal/agent/agent.go:931-941` — replaces `var maxOutputTokens int64 = 40`
with `maxOutputTokens := smallTitleMaxTokens`. The `CanReason` branch is
preserved so models correctly marked reasoning still get the larger
catwalk-defined cap.

`internal/agent/agent.go:976-985` — inside the existing "small failed → try
big" path, the big-model `newAgent` call now uses
`cmp.Or(largeModel.CatwalkCfg.DefaultMaxTokens, fallbackTitleMaxTokens)`
instead of inheriting the small model's `maxOutputTokens`. This is a real
latent bug fix beyond the empty-text retry.

`internal/agent/agent.go:994-1010` — new "empty visible text" retry block,
guarded by `model.CatwalkCfg.ID != largeModel.CatwalkCfg.ID` so a same-model
retry is skipped, and by `isUsableTitleResponse`. On failure or another empty
response it falls through to the existing default-name path.

`internal/agent/agent.go:1069-1079` adds:

```go
func isUsableTitleResponse(resp *fantasy.AgentResult) bool {
    if resp == nil { return false }
    return strings.TrimSpace(resp.Response.Content.Text()) != ""
}
```

Doc typo `"session titled"` → `"session title"` and slog key `"err"` →
`"error"` normalization on the touched lines.

## Strengths

- Constants are named and documented, not magic numbers. The `cmp.Or` for the
  large-model fallback cap is the right primitive here.
- The same-model guard (`model.CatwalkCfg.ID != largeModel.CatwalkCfg.ID`)
  cleanly avoids the "small == large == same model id" pointless retry that
  would otherwise hit the same provider call twice.
- The latent-bug callout in the description (large model inheriting the
  small-model cap) matches the diff exactly.
- The trade-offs section in the PR body is unusually candid — explicit about
  why `finish_reason` was rejected as a signal and why no regression test
  ships with the patch.

## Concerns

- The new retry runs *after* the existing "small failed → big retried" path
  has already executed. On a flow where the small model errored AND the big
  model then *also* returned empty text, we currently retry the big model a
  second time with the same params — wasted call. Consider tracking whether
  the large model has already been tried this turn and skipping the second
  attempt when so.
- `isUsableTitleResponse` calls `strings.TrimSpace` on
  `resp.Response.Content.Text()` — make sure `Text()` is safe to call on a
  `nil` Content (the nil-resp branch handles `resp == nil`, but not a
  populated resp with nil content). Worth a quick defensive check.
- No regression test. The PR body acknowledges this and defers to a VCR
  cassette + provider OAuth wiring. A pure unit test that injects a stub
  `fantasy.AgentResult` with empty text would have been very cheap and would
  guard the `isUsableTitleResponse` branch directly without touching network.

## Nits (non-blocking)

- The slog key normalization (`err` → `error`) is great but only happens on
  touched lines; the rest of the file still mixes the two. Optional sweep
  follow-up.
- Comment "Welp, the large model didn't work either" is unchanged but is now
  slightly misleading because there's a *second* large-model attempt below.

## Verdict

**merge-after-nits** — solid fix; would like the double-retry edge case
addressed and at least a unit test for `isUsableTitleResponse` before
merging.
