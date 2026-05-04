# charmbracelet/crush#2741 — fix(styles): fix some regressions where colors were incorrect

- PR ref: `charmbracelet/crush#2741` (https://github.com/charmbracelet/crush/pull/2741)
- Head SHA: `2a428a6666e2ba7f5e5f1ed6ca6ed8052abd75ce`
- Title: fix(styles): fix some regressions where colors were incorrect
- Verdict: **merge-as-is**

## Review

Eight near-identical fixes in `internal/ui/styles/quickstyle.go`, all swapping the
foreground from `o.primary` to `o.onPrimary` on widgets where the background is one of
`o.error`, `o.destructive`, `o.info`, `o.infoMoreSubtle`, `o.secondary`, or `o.primary`.
Concrete sites: `quickstyle.go:245` (Error primitive), `:621` (`Tool.ErrorTag`), `:626`
(`Tool.NoteTag`), `:639` (`Tool.AgentTaskTag`), `:687` (`Button.Focused`), `:786`
(`Messages.ErrorTag`), `:850` (`Dialog.SelectedItem`), `:911` (`Status.ErrorMessage`),
and `:915` (`Completions.Focused`).

Every one of these had the same latent bug: putting `o.primary` foreground over a
non-`primary` background gave you primary-on-saturated, which on most themes is
illegible (e.g., red-on-red for the error tag, primary-on-primary for the completions
focus state — the last one literally rendered text invisible since both fg and bg were
`o.primary`). `o.onPrimary` is the canonical "text designed to sit on top of primary
surfaces" token, so the substitution is exactly the contrast token these widgets were
always meant to use.

The change at `quickstyle.go:911` (`Completions.Focused`) is the most important — that
was the invisible-text regression. The change at `:687` (`Button.Focused`) is the
second most user-facing. Worth a quick visual smoke on the dialog list selection
(`:850`) since `Dialog.SelectedItem` keeps `o.primary` as background and the foreground
shifts from `o.fgBase` to `o.onPrimary` — this is the only site where the old
foreground wasn't `o.primary`, so behavior on themes where `fgBase` and `onPrimary`
diverge (e.g., themes that override one but not the other) will look different. Should
still be net-better. No nits.
