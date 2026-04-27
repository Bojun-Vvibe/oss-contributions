# PR #24645 — tui: ignore invalid custom themes to prevent startup crashes

- **Repo**: sst/opencode
- **PR**: #24645
- **Head SHA**: `43f18a1a43599b370d354e57250552bb5a1430bb`
- **Author**: thdxr (Dax)
- **Size**: +2 / -1 across 1 file
- **Verdict**: **merge-as-is**

## Summary

Two-line guard in `getCustomThemes()` that runs each user-supplied
theme JSON through the existing `isTheme(...)` validator before
landing it in the `result` map. Without this, a malformed local
file under the theme directory crashes the TUI at startup because
downstream code assumes every entry of the returned record has
`scheme`/`palette`/etc.

## Specific changes

- `packages/opencode/src/cli/cmd/tui/context/theme.tsx:500-504`
  — read the JSON into a local `theme` first, then assign to
  `result[name]` only when `isTheme(theme)` returns true. The
  parse path itself (`Filesystem.readJson`) already throws for
  syntactically broken JSON; this PR closes the *valid JSON, wrong
  shape* gap (e.g. `{}`, `{ "name": "x" }`, partial drafts).

## Risks

- **Silent skip**: invalid themes disappear from the picker
  without any log line. For a startup-blocker fix that's the right
  default (don't crash, don't spam), but a one-line `log.warn`
  with the offending filename would help users debug "why isn't
  my theme showing up". Not a blocker — can ship as a follow-up.
- **No test**: the diff doesn't add a regression fixture. The fix
  is two lines and the validator is already covered, so the
  marginal value of a dedicated test is low; the bigger payoff
  would be a snapshot test against `getCustomThemes()` with a mix
  of valid + invalid + empty-object inputs to pin the behavior in
  case the validator gets refactored.

## Verdict

`merge-as-is` — straightforward defensive fix in the only path
that matters (TUI boot). The `isTheme` predicate is the right
validator to delegate to, and the early-skip pattern is consistent
with how the rest of the file handles missing/optional inputs.

## What I learned

The classic "user-supplied JSON is structurally valid but
semantically wrong" failure mode has only one safe shape at a
startup-critical entry point: validate-then-include, never
include-then-pray. The fact that the fix is two lines and the
validator already existed is the giveaway — the original code
just trusted the file system, and that trust was load-bearing for
the whole TUI bootstrap.
