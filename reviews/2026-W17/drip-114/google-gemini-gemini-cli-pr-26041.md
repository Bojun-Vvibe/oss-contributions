# google-gemini/gemini-cli PR #26041 — fix(cli): replace ambiguous width characters with halfwidth alternatives

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26041
- **HEAD**: `8ae76976`
- **Touch**: ~50+ files across `packages/cli/src/`, mostly tests and
  snapshots; +/− is largely symmetric

## Context

CJK terminals (CN/JP/KR locales, plus VS Code's integrated terminal in
those locales, plus tmux/iTerm2 with East-Asian-width set to Ambiguous
= Wide) render Unicode "ambiguous width" characters at 2 columns
instead of 1. The `→` U+2192 RIGHTWARDS ARROW that the codebase uses
liberally in user-visible strings (`Extension "X" successfully updated:
1.0.0 → 1.1.0.`) and in test fixtures/snapshots was rendering as 2
columns there, breaking right-aligned columns in TUI panels and
producing snapshot drift between dev environments.

## Diff

Mass substitution of `→` (U+2192, ambiguous-width per UAX #11) with
`￫` (U+FFEB, HALFWIDTH RIGHTWARDS ARROW, definitionally narrow). At
real call sites:

- `packages/cli/src/commands/extensions/update.ts:31/93` — the
  `Extension "${name}" successfully updated: ${old} ￫ ${new}.`
  log string and the matching `expectedLog:` test string in
  `update.test.ts:128/170`.
- `packages/cli/src/acp/acpResume.test.ts:275/294` — comment-only
  changes (`Successful tool call ￫ 'completed'`).
- All the affected snapshots (`__snapshots__/*.snap`,
  `__snapshots__/*.snap.svg`) get updated in the same commit so CI
  stays green.

The substitution is mechanically correct: `￫` *is* East-Asian-narrow
and renders 1-column in every terminal that respects the spec.

## Risks / nits

- **Visual regression on non-CJK terminals.** `￫` U+FFEB is a halfwidth
  arrow originally designed to fit Japanese halfwidth-katakana columns;
  in Latin-1 fonts it renders noticeably narrower and slightly higher
  than the surrounding glyphs (especially in monospace fonts that don't
  ship the U+FFEx halfwidth block — many fall back to a placeholder).
  For US/EU users on the default terminal font this will look worse,
  not better. The right fix is usually `->`  (ASCII) for log lines
  and `\u2192` only in places where the visual arrow really matters
  with an explicit width-aware layout. Recommend the team consider
  `->` as the no-regret target.
- **Snapshot churn dilutes review value.** The diff is ~3800 lines
  but ~98% is `.snap` / `.snap.svg` updates. Hard to spot any *real*
  change buried in the substitution. A targeted commit splitting
  "source-only changes" from "snapshot regeneration" would be much
  more reviewable, and would also let the snapshot regeneration be
  re-run mechanically rather than reviewed line-by-line.
- The replacement also touches comment-only sites
  (`acpResume.test.ts:275/294`'s `// Successful tool call ￫
  'completed'` comments) — those are safe but unnecessary; comments
  don't render in any terminal. Keeping them at `→` reduces churn.
- No CI lint added to *prevent reintroduction*: the next PR that
  copy-pastes from a doc / Slack message will reintroduce `→` and
  the cycle repeats. A `no-ambiguous-width-glyphs` ESLint rule (or a
  pre-commit grep over `[→←↑↓➜⇒⇨]`) would close the loop.
- The `→` U+2192 is not the only ambiguous-width arrow in source
  (check `→`, `↑`, `↓`, `←`, `⇒`, `▸`, `…`, `°`, `±`, `×`, `÷`); the
  PR title says "characters" plural but the substitution appears to
  be only `→`. Either expand the scope and rename, or rename to
  "replace `→` with halfwidth alternative".

## Verdict

Verdict: needs-discussion
