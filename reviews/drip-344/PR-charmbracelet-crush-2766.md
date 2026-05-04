# charmbracelet/crush#2766 — feat(hyper): show remaining hypercredits in the sidebar

- PR ref: `charmbracelet/crush#2766`
- Head SHA: `0efaca24ef5cef1f21e57f21cfa193d65df57c39`
- Title: feat(hyper): show remaining hypercredits in the sidebar
- Verdict: **merge-after-nits**

## Review

Functionally clean — adds a `FetchCredits` HTTP client in
`internal/agent/hyper/provider.go:65-97` that hits `/v1/credits`, plumbs the result
through a new `creditsUpdatedMsg` Bubble Tea message (`internal/ui/model/ui.go:155-158`),
and renders it in the sidebar via the new `hyperCredits *int` parameter on
`common.ModelInfo` (`internal/ui/common/elements.go:43`,`:81-86`). The pointer-typed
`*int` instead of `int` is a deliberate "unknown vs zero" distinction — good call,
since 0 credits is a meaningful state distinct from "haven't fetched yet."

The fetch is triggered both at `Init()` when the active provider is hyper
(`ui.go:398-400`) and after every model selection that lands on a hyper model
(`ui.go:1700-1702`), which matches the PR title's "after each prompt" comment in the
struct doc — though strictly speaking there's no per-prompt refresh, only per-model-
select. The doc comment at `ui.go:273` slightly overstates the cadence; either tighten
the comment or add a refresh on prompt-completion.

Nits:

1. `formatCredits` in `internal/ui/common/elements.go:128-148` is a hand-rolled
   thousands separator. `golang.org/x/text/message` (`message.NewPrinter(language.English).Sprintf("%d", n)`)
   would do this in one line and respect locale; not worth blocking on but worth
   replacing later.
2. `FetchCredits` (`provider.go:67-97`) hardcodes a `10 * time.Second` client timeout
   *and* the caller wraps it in another `10 * time.Second` `context.WithTimeout`
   (`ui.go:1612`). The inner timeout is redundant — drop one or document why both
   exist. Prefer keeping only the context timeout so callers control deadline.
3. The `slog.Error` on fetch failure (`ui.go:1617`) is too loud for a non-essential
   sidebar widget; transient credit-API outages will spam logs. Demote to
   `slog.Debug` or `slog.Warn`.

None of these are merge-blockers; ship after addressing the timeout duplication at
minimum.
