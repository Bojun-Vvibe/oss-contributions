# PR #26042 — fix(cli): include active prompt in copy mode history

- **Repo**: google-gemini/gemini-cli
- **PR**: #26042
- **Head SHA**: `82068989eb4bb47ec08559dd891336523483dc60`
- **Author**: Ultron09 (Suryaansh Prithvijit Singh)
- **Size**: +465 / -428 across ~95 files
- **Verdict**: **request-changes**

## Summary

PR title and body describe a small, well-motivated UX fix: when
the user enters copy mode (F9), the currently-typed prompt is not
selectable in the scrollable history because it lives in the
static composer pane below the virtualized list. The fix is to
inject a synthetic `HistoryItem` for the active prompt at the
end of the history list while copy mode is active and to hide
the static `Composer` so the full terminal height becomes
selectable. That part of the diff (in
`MainContent.tsx:+43/-6` and `Composer.tsx:+3/-3`) is the right
shape and reuses `HistoryItemDisplay` for visual consistency.

## Specific changes (the legitimate fix)

- `packages/cli/src/ui/components/MainContent.tsx:+43/-6` —
  builds an `augmentedHistory` derived from `historyItems` and,
  when `inputState.copyModeEnabled` is true, appends a synthetic
  history entry representing the live prompt buffer. Memoization
  key includes `[augmentedHistory, historyItems,
  inputState.copyModeEnabled]`, which is correct: the synthetic
  entry must invalidate when either the source list or the
  copy-mode flag changes.
- `packages/cli/src/ui/components/Composer.tsx:+3/-3` — gates
  the static composer block on `!copyModeEnabled` so it
  collapses to nothing in copy mode (`height={copyModeEnabled ?
  uiState.stableControlsHeight : undefined}` retains the layout
  reservation), making the freed terminal rows available to the
  scrollable list.

## Why this is `request-changes`

The remaining ~430 lines of churn are **not** about copy mode.
The PR performs a destructive global character substitution
across 80+ unrelated files, replacing the U+2192 Rightwards Arrow
(`→`) with U+FFEB Halfwidth Rightwards Arrow (`￫`). Examples:

- `packages/cli/src/commands/extensions/update.ts:32` — user-facing
  log line `"successfully updated: ${old} → ${new}"` becomes
  `"... ${old} ￫ ${new}"`.
- `packages/cli/src/commands/extensions/update.test.ts:128, 170` —
  regression assertions are *updated* to expect the halfwidth
  glyph, locking in the visual change.
- `packages/cli/src/commands/gemma/setup.ts:180`,
  `gemma/status.ts:131` — user-facing platform/routing copy.
- `packages/cli/src/services/FileCommandLoader.ts:79, 146` —
  *code comments* changed from `User → Project → Extension` to
  `User ￫ Project ￫ Extension`.
- `packages/cli/src/config/policy-engine.integration.test.ts:556,713`
  — *test comments* about priority arithmetic.
- `packages/cli/src/ui/__snapshots__/App.test.tsx.snap` and ~40
  other snapshot files — selected-row marker changes from
  U+25CF Black Circle (`●`) to U+2219 Bullet Operator (`∙`) in
  rendered SVG snapshots, plus the arrow swap.

That's a pile of red flags:

1. **Wrong glyph for the use case.** U+FFEB (Halfwidth Rightwards
   Arrow) is part of the CJK Halfwidth Forms block; rendering
   support is far worse than U+2192, especially in non-monospace
   contexts and on Windows terminals where the original `→`
   already had known issues that #26041 (a sibling PR) is trying
   to fix properly. Swapping to a less-supported character
   silently degrades non-target users.
2. **Scope leak.** PR title is "include active prompt in copy
   mode history". The Unicode swap is invisible to a reviewer
   reading only the title and is exactly the class of change that
   should be its own PR with its own justification.
3. **Snapshot tests updated to *match* the change rather than
   *reject* it.** That's a code smell — when a refactor's
   regression suite is "rewrite the goldens", the regression
   suite has stopped being a guard. Either the original `→` /
   `●` characters were the contract (and the change is wrong),
   or they weren't (and the snapshots shouldn't have included
   them). Either way, the PR doesn't make the case.
4. **`packages/cli/src/ui/components/messages/DenseToolMessage.tsx:+9/-9`
   plus its test file +17/-17 plus an SVG snapshot** — that's a
   message-component rewrite tucked into the same diff. Even if
   it's just glyph swaps, it touches a hot rendering path that
   deserves its own PR for `git blame` reasons alone.

## Risks

- **Production text regression for every user that doesn't have
  good U+FFEB rendering** — most Windows console fonts, many
  SSH terminals, and the in-IDE terminal panes of several
  popular editors fall back to `?` or a tofu glyph for
  Halfwidth-Forms code points. The legitimate copy-mode fix
  ships a regression along with it.
- **Snapshot drift cost** — the goldens-rewrite pulls 40+ SVG
  snapshot updates into this PR's blame, making future
  bisection across copy-mode behavior difficult.
- **Cross-PR conflict** — sibling PR #26041 (titled
  "replace ambiguous width characters with halfwidth alternatives")
  is the *correct home* for this kind of glyph migration and
  is being reviewed on its own merits. Two PRs touching the
  same character set with different rationales is a recipe for
  rebase pain and inconsistent treatment.

## Verdict

`request-changes` — split this into two PRs:

1. **The actual copy-mode fix**: `MainContent.tsx`,
   `Composer.tsx`, the synthetic-history injection logic, and
   the (currently absent) tests asserting the synthetic item
   appears at the tail of the augmented history when
   `copyModeEnabled` flips true and disappears when it flips
   false. Should land as ~50-80 lines, including a regression
   test for the explicit-prompt-wins ordering.
2. **The Unicode normalization**: rebase against / coordinate
   with #26041 so there is one source of truth for the project's
   glyph policy. Make the case for U+FFEB vs U+2192 explicitly,
   document the Windows-rendering motivation, and update the
   snapshot baselines as a separate, reviewable change.

The copy-mode fix on its own is `merge-after-nits` material; the
bundled Unicode swap is what pushes the whole PR to
`request-changes`.

## What I learned

A PR scope leak that *also* updates the regression goldens to
match the leaked change is the most expensive kind: the goldens
were the rejection mechanism, and the PR has disarmed it. When
reviewing any change that touches snapshot files for unrelated
rendering reasons, the question to ask first is "did the
snapshots *force* the author to acknowledge a behavioral change,
or did the author *teach* the snapshots a new behavior?" If the
latter, either the snapshots are too brittle (own-PR problem) or
the change is sneaking past the guard (this-PR problem).
