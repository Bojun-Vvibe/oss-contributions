# sst/opencode PR #24507 — fix: make scrollbar visible in Monokai theme

- Repo: sst/opencode
- PR: #24507
- Head SHA: `ff3b9055`
- Author: @alfredocristofano
- Diff: targeted +2/-2 on `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1067-1068` (plus the same unrelated 100-file import-migration drag-along seen in #24502/#24503/#24504)
- Closes: #24425

## What changed

In the session view's scrollbar `trackOptions`, swaps:

- `backgroundColor: theme.backgroundElement` → `theme.background`
- `foregroundColor: theme.border` → `theme.borderActive`

In Monokai both `backgroundElement` and `border` resolve to `#3e3d32`, which made the thumb invisible against its own track. The fix uses the same pair already in `sidebar.tsx` and `permission.tsx`, where the thumb is drawn in `borderActive` (cyan in Monokai) over the main `background`.

## Specific observations

- The fix is the right pattern — the bug is a theme-token collision, not a Monokai-specific patch — so aligning with the cross-route convention `(theme.background, theme.borderActive)` rather than adding a one-off Monokai override is the correct move. Verified the same pair is the convention by the contributor's reference to `sidebar.tsx` / `permission.tsx`.
- Worth a maintainer skim to make sure no other theme has `background == borderActive` collision (the original bug would just shift to a different theme). A mechanical check in the theme registry (`theme.background !== theme.borderActive` invariant test) would catch this class of bug going forward — a one-test follow-up, not a blocker for this PR.
- Same diff-hygiene pattern as other PRs in this batch from the same author (#24502, #24503, #24504): PR drags AGENTS.md rewrite + a `Config` import-path migration across ~30 files. The targeted scrollbar fix is two lines; the rest is noise from a stale rebase. Ideally split.

## Verdict

`merge-after-nits`

## Rationale

The targeted change is correct, minimal, and consistent with the existing convention in sibling components. Diff hygiene is the only nit; the actual scrollbar fix is sound.
