# PR #8946 — fix goose2 window minimum sizing

- Repo: block/goose
- Head: `ff4106c5390de1c198fd36a807fa297af6d8702b`
- URL: https://github.com/block/goose/pull/8946
- Verdict: **merge-as-is**

## What lands

Two coordinated changes fixing the case where the goose2 desktop window can
be resized so small that the settings UI overlaps the macOS traffic-light
window controls and clips essential content:

1. **Tauri window minimum**: `ui/goose2/src-tauri/tauri.conf.json:21-23` adds
   `"minWidth": 800, "minHeight": 600` to the window config — matches the
   existing launch dimensions (`width: 800, height: 600` on lines 19-20),
   which is the right floor: there's no point allowing the window to shrink
   below the smallest size the shell was ever designed for.

2. **Settings modal responsive sizing**:
   `ui/goose2/src/features/settings/ui/SettingsModal.tsx:155-165` swaps the
   prior fixed `h-[600px] w-full max-w-3xl` for
   `h-[min(600px,calc(100vh-4rem))] w-[calc(100vw-2rem)] max-w-3xl`. This
   means:
   - Height: prefer 600 px, but cap at viewport height minus 4rem so the
     modal can never exceed the window with chrome accounted for.
   - Width: viewport width minus 2rem (1rem margin per side), still capped by
     `max-w-3xl` so it doesn't get absurdly wide on large displays.

3. **Settings nav scrolls independently**:
   `SettingsModal.tsx:165-205` wraps the sidebar nav in
   `<nav className="min-h-0 flex-1 overflow-y-auto px-2 pb-3">` with the
   `<div className="flex flex-col gap-1">` *inside* it, plus
   `flex-shrink-0` on the parent sidebar at `:165`. This is the load-bearing
   piece: with the new height cap the nav could otherwise be the thing that
   pushes content off-screen, so giving it its own scroll surface (with
   `min-h-0 flex-1` so the flexbox actually permits the scroll) keeps the
   modal usable at the floor size.

## Why merge-as-is

- The Tauri minimum and the settings modal sizing are correctly coupled: the
  modal calculations (`100vh - 4rem`, `100vw - 2rem`) are guaranteed to leave
  positive space because the window itself can't shrink below 800x600.
- The `min-h-0 flex-1 overflow-y-auto` pattern on the nav is the correct
  Tailwind/flexbox idiom for "let me scroll inside a flex container" — without
  `min-h-0` flex children default to `auto` which prevents inner scroll. The
  fix author got this right.
- `flex-shrink-0` on the sidebar prevents the navigation column from
  collapsing to zero width when the modal hits its width floor — preserves
  the navigation usability at the smallest allowed size.
- Reviewer `kalvinnchau` already approved.
- Verification list in the PR body is honest: jq lint, biome, pnpm build,
  cargo fmt + clippy + check, plus visual confirmation from the original
  reporter.

Small surface (31 / -27 across 2 files), correct fix at both layers (config
floor + responsive modal), no test surface to add (purely visual/layout). Ship.