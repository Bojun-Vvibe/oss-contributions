# block/goose #9011 — polish inline code snippet styling

- **Head SHA:** `ad0c9f63a90e569882db944c8e9dd0d4b619f45d`
- **Base:** `main`
- **Author:** morgmart
- **Size:** +12 / −0 across 1 file
  (`ui/goose2/src/shared/styles/globals.css`)
- **Verdict:** `merge-as-is`

## Summary

Adds 12 lines of CSS targeting `[data-streamdown] :not(pre) > code` —
i.e. inline `code` spans rendered inside the `streamdown` markdown
component, but *not* code inside `pre` blocks. Gives them a soft
border, slight padding, muted background, and a smaller font weight so
inline code is visually distinct from surrounding prose without
clashing with full code blocks (which have their own block-level
styling elsewhere).

## What's right

- **Selector specificity is exactly right.** `[data-streamdown]
  :not(pre) > code` (`globals.css:843`) scopes the rule to the
  streamdown subtree only (so it doesn't leak to other markdown
  renderers in the app), and `:not(pre) > code` ensures the styling
  only hits inline code — `pre > code` (the block-level case) is
  left untouched. Without the `:not(pre)` guard, the soft border
  would have been drawn around every code-block line, which would
  look terrible.

- **Padding values are sensibly small.** `0.08rem 0.28rem` (line 846)
  — vertical padding nearly zero so inline `code` doesn't push line
  height around in flowing prose, with just enough horizontal padding
  to separate the bordered chip from adjacent words. Anyone who's
  styled inline code knows that vertical padding > 0.15rem causes
  visible line-height drift.

- **`color-mix(in oklab, var(--background-muted) 72%, transparent)`
  for the background** (line 845) is the modern correct approach:
  uses the design-token background, mixes 72% of it with transparency,
  in OKLab for perceptually uniform mixing. This means the rule
  adapts to both light and dark themes automatically (since
  `--background-muted` is theme-defined) without needing two
  variants. Browser support: `color-mix()` is in all evergreen
  browsers as of mid-2024 — fine for an Electron-based desktop app.

- **`font-size: 0.86em` and `font-weight: 500`** (lines 847-848)
  shrink inline code slightly relative to surrounding prose and bump
  weight from 400 → 500. Standard typographic move to make
  monospace-vs-proportional differences less jarring at the same
  visual size.

- **`vertical-align: baseline`** (line 850) is the explicit fix for
  the common bug where browsers default inline `code` to a slightly
  different baseline than the surrounding text, causing visible
  vertical jitter in mixed prose+code lines.

- **Uses existing design tokens** (`--border-soft`,
  `--background-muted`, `--foreground`) rather than hardcoded colors
  — so the rule won't break when the design system updates its
  palette. This is exactly the pattern the rest of `globals.css`
  follows.

- **Diff is purely additive.** No deletions, no modification of
  existing rules. The new block is placed in a sensible spot (right
  before the `goose2 custom utilities` section header at line 853),
  so the file structure stays predictable.

## Risk

- **Visual regression risk: zero for non-`[data-streamdown]`
  markdown.** The selector is so tightly scoped that nothing outside
  streamdown's render tree can be affected.
- **Visual regression risk inside streamdown: low.** The change is
  purely additive (inline `code` previously had no specific styling
  — just whatever browser default + parent font), so the worst case
  is "looks slightly different than before" rather than "previously
  styled element now has conflicting styles".
- **Theme contrast:** `color-mix(... 72%, transparent)` against a
  light theme could produce a very subtle background that doesn't
  visually distinguish the `code` chip from prose. Worth eyeballing
  on light theme before merge — but if the design tokens are
  well-chosen this should already work.

## Nits

None worth blocking on.

- The CSS could optionally add `text-shadow: none` to defend against
  any inherited text-shadow, but in practice no Block UI surface
  applies text-shadow to body text.

## Verdict

`merge-as-is`. Purely visual polish, well-scoped selector, uses
design tokens, additive only. Boring and correct.
