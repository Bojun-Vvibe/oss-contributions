# sst/opencode #25700 — feat: add copy_on_select tui config to control auto-copy behavior

- **Head SHA reviewed:** `8145a313426a79339739643808570b83c4a72d78`
- **Size:** +21 / -5 across 4 files
- **Verdict:** request-changes

## Summary

Promotes the previously env-var-only `copy_on_select` toggle into a
proper `tui.json` config option, and slips in a second new option
(`message_click_actions`) along the way.

## What I checked

- `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts:27-28` adds
  two new keys to `TuiOptions`. The `copy_on_select` description says
  "Default: false on Windows, true on macOS/Linux" — but the actual
  behavior implemented in `app.tsx` does **not** match that. The new
  helper `disableCopyOnSelect()` returns `false` whenever the user did
  not set the key, which means the *previous* env-var-only path
  (where macOS/Linux already auto-copied via `onMouseUp`) is now the
  default for *all* platforms including Windows. That's a behavioral
  change, and the schema docstring is misleading.
- `packages/opencode/src/cli/cmd/tui/app.tsx:263-269` — the helper is
  named `disableCopyOnSelect` but reads as "should we treat the env
  flag as set?". The semantics are inverted vs. the name: when
  `tuiConfig.copy_on_select === true`, the helper returns `false`,
  which then makes `useKeyboard` `return` early (skip the disable
  branch). It works, but the inversion is a foot-gun for the next
  reader. Rename to `copyOnSelectDisabled` or invert the truth table.
- `packages/opencode/src/cli/cmd/tui/ui/dialog.tsx:159-164` duplicates
  the same helper inside `DialogProvider`. The two definitions can
  drift; lift it into a shared util (e.g.
  `@tui/util/copy-on-select.ts`) and import from both call sites.
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1152` —
  `message_click_actions` is added here as an opt-out for the
  Revert/Copy/Fork popup, but the schema description says
  "Default: true" while the *actual* default in code is also true
  (because the guard is `=== false`). The PR title and body do not
  mention this option at all; surface it in the PR description so
  reviewers know it's part of the contract.

## Risk

Low-to-medium. The PR ships a behavior change disguised as a config
addition: pre-PR, Windows users had `OPENCODE_EXPERIMENTAL_DISABLE_
COPY_ON_SELECT=1` baked in by build-time guard (or platform default in
the surrounding code, depending on rev). Post-PR, the platform default
is collapsed to a single non-platform-aware path. Worth re-reading the
old default code to confirm no regression for Windows users who never
touch tui.json.

## Recommendation

Request changes:

1. Make the schema docstring match runtime behavior, or restore the
   platform-aware default by checking `process.platform === "win32"`
   in `disableCopyOnSelect()` when `copy_on_select` is undefined.
2. Rename `disableCopyOnSelect` so its return value matches its name,
   and lift it to a shared util used by both `app.tsx` and
   `dialog.tsx`.
3. Mention `message_click_actions` in the PR body — it's a separate
   user-facing option and deserves its own line in the changelog.
4. Add a unit test on the helper's truth table (env flag set, config
   true, config false, config undefined × platform) so the matrix is
   pinned.
