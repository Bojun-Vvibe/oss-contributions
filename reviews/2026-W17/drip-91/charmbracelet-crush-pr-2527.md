---
pr: 2527
repo: charmbracelet/crush
sha: c4f3ac51b7e94bab043938e4d5eae2bb56062996
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2527 — feat(ui/dialog): add mouse click handling for permission buttons and compute hitboxes

- **Head SHA**: `c4f3ac51b7e94bab043938e4d5eae2bb56062996`
- **URL**: https://github.com/charmbracelet/crush/pull/2527
- **Size**: small (106/2 across 2 files)

## Summary
Adds left-click support to the permission dialog ("Allow" / "Allow for Session" / "Deny") by computing per-button hitboxes during `Draw()` and matching mouse clicks against them in `HandleMsg()`. Also fixes a latent bug in `internal/ui/model/ui.go` where mouse events delivered to dialogs returned the dialog's command but discarded its `tea.Cmd`.

## Specific findings
- `internal/ui/dialog/permissions.go:80-83` — adds `buttonsHitboxesValid bool` and `buttonsHitRects [3]image.Rectangle`. The `Valid` flag is the right shape: it ensures clicks before the first paint can't accidentally hit a stale `image.Rectangle{}` (which would be `(0,0)-(0,0)` and `pt.In(...)` is false anyway, but the explicit guard at `:282` is good defensive code).
- `permissions.go:281-296` — the click dispatch matches `msg.Button == tea.MouseLeft` only. Right tone for a permission dialog: middle/right click should not accidentally grant a permission. The branches set `p.selectedOption` before calling `respond()`, so any code reading `selectedOption` from a downstream message sees the right value.
- `permissions.go:462-541` — hitbox computation duplicates the layout math from the actual `lipgloss` render: `buttonTopInInner := headerHeight + 1`, `if contentIncluded { buttonTopInInner += contentHeightActual + 2 }`, then horizontal-vs-stacked branching keyed on `lipgloss.Width(buttonGroupHorizontal) > contentWidth`. **This is the main risk surface in the PR.** Any future change to the inner layout (extra blank line between content and buttons, header height shift, etc.) silently desyncs the hitboxes from the rendered buttons. Suggest extracting a single helper that returns both the rendered string and the hitboxes from one walk over the layout, instead of recomputing the offsets twice.
- `permissions.go:512-525` — `buttonsAreStacked` calculation uses the horizontal group's width vs `contentWidth`; matches the actual stacking trigger in `Draw()`. The vertical loop adds one button height per iteration (`y += h`), which is correct since each `Button(...)` is single-row in the current style but would silently break if button styles ever wrap.
- `permissions.go:533-541` — horizontal branch positions buttons right-aligned (`x := innerMinX + (contentWidth - buttonGroupWidth)`). That matches `ButtonGroup`'s right-justified layout. The `hitboxPad := 1` gives a 1-cell-wide forgiveness border, which is generous on terminals where mouse cells already round; not unreasonable, but worth a comment that it intentionally overlaps adjacent button hitboxes by 2 cells (1 from each side) — `image.Rectangle.In` resolves overlap by first-match in the switch, so `Allow` wins ties with `Allow for Session`. Document or reduce to 0.
- `internal/ui/model/ui.go:677-680` — the change from `m.dialog.Update(msg)` to `m.handleDialogMsg(msg)` is unrelated to mouse hitboxes but is a real bug fix: the previous form was discarding the returned `tea.Cmd` so any dialog-initiated effect (like `respond()`'s callback) would silently never run on mouse input. Worth calling out in the commit message — currently it's hidden inside this PR.

## Risk
Low. Hitboxes are rebuilt every frame; stale hitboxes can't outlive the dialog. The duplicated layout math is the main maintenance hazard.

## Verdict
**merge-after-nits** — extract a single layout-walk helper to remove the duplicated math (or at least add a comment cross-referencing the two locations), and split or call out the `ui.go` mouse-cmd-discard fix in the commit message since it's a separate bug.
