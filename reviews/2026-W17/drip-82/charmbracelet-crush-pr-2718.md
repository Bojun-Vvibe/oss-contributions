# charmbracelet/crush PR #2718 — chore(ui): theme prep

- Repo: charmbracelet/crush
- PR: #2718
- Head SHA: `d5f5dbd6`
- Author: @meowgorithm
- Diff: +1027/-943 across 10 files

## What changed

Renames every call to `styles.DefaultStyles()` over to `styles.CharmtonePantera()` across the UI surface (`internal/app/app.go`, `internal/cmd/run.go`, `internal/cmd/session.go`, `internal/ui/chat/tool_result_content_test.go`, `internal/ui/common/common.go`, `internal/ui/logo/example/main.go`, and likely four more behind the `+1027/-943` envelope). Title says "theme prep" — the implication is `DefaultStyles` is being deprecated in favor of named-theme constructors so a future "theme picker" feature can swap palettes by string name.

## Specific observations

- The mechanical rename at `internal/cmd/session.go:441` shadows the imported `styles` package with a local `styles` variable: `styles := styles.CharmtonePantera()` then a few lines later `items := chat.ExtractMessageItems(&styles, msg, toolResults)` at `:482`. This compiles but is confusing — every subsequent reference to the package `styles.*` in the same function body is now the local variable. The previous code used `sty := styles.DefaultStyles()` which avoided the shadow. Keep the `sty` (or rename to `theme`) variable name; don't shadow the package.
- The PR title `chore(ui): theme prep` understates the change. `DefaultStyles()` was the API; renaming every consumer to a hardcoded `CharmtonePantera()` removes the indirection that "theme prep" implies should be added — if the goal is theme switching, the right shape is a `styles.For(themeName string)` factory or `styles.Default()` returning a `*Theme` interface that points at Pantera by default. Hardcoding `CharmtonePantera` at every call site means a future "theme picker" PR will have to do this exact same 10-file rename a second time.
- The `internal/app/app.go:236` and `internal/cmd/run.go:192` non-interactive spinner branches still call `hasDarkBG := true` immediately after the new `styles.CharmtonePantera()` call — but the whole point of having multiple themes is that not every theme is dark-default. The light-background detection from drip-80's #2538 wires through `applyBackgroundTheme` for the interactive path; the non-interactive path should also be calling `DefaultStylesForBackground(bool)` (also from #2538) instead of unconditionally calling `CharmtonePantera()`.

## Verdict

`needs-discussion`

## Rationale

The rename is mechanically clean but the design direction is unclear: "theme prep" with hardcoded constructor calls at every site doesn't actually prepare for theme switching, it postpones the same refactor. Recommend the maintainer thread a design discussion on whether the desired end state is (a) `styles.For(name string) *Theme`, (b) a `Theme interface` with `CharmtonePantera` as one impl, or (c) keep `DefaultStyles()` as the single entry and have it dispatch internally. Also resolve the package-shadowing in `cmd/session.go` and align the non-interactive paths with the `DefaultStylesForBackground` API from #2538 before merge.
