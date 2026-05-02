# Review — google-gemini/gemini-cli#26324

- PR: https://github.com/google-gemini/gemini-cli/pull/26324
- Title: fix(cli): prevent ghost text wrapping infinite loop
- Head SHA: `477bfde85dfc12867690ccbc1d157c87e94db081`
- Size: +57 / −20 across 2 files
- Verdict: **merge-as-is**

## Summary

The previous word-wrap loop (`while (stringWidth(wordToProcess) >
inputWidth) { … }`) could spin forever when `inputWidth <= 0` (terminal
collapsed to zero width) or when a single grapheme's display width
exceeded `inputWidth` (CJK char in a 1-col window). The PR extracts the
splitter into a pure `splitWordForWidth(word, maxWidth)` function that
clamps `maxWidth` to a positive lower bound and forces progress on every
character.

## Evidence / specific spots

- `packages/cli/src/ui/components/InputPrompt.tsx`:
  - New exported `splitWordForWidth` (lines 163-191): `const width =
    maxWidth > 0 ? maxWidth : 1;` clamps; the loop appends every
    `char` and flushes when `currentPartWidth + charWidth > width &&
    currentPartWidth > 0`, then re-flushes when a single char alone
    exceeds `width`. Both branches guarantee forward progress.
  - The render path (lines 1518-1521) is reduced to `const splitWord
    = splitWordForWidth(word, inputWidth); additionalLines.push(
    ...splitWord.slice(0, -1)); currentLine = splitWord.at(-1) ?? '';`
- `packages/cli/src/ui/components/InputPrompt.test.tsx`:
  - Three new tests (lines 486-505) cover `maxWidth = 0`, wide
    characters exceeding `maxWidth = 1`, and `Infinity` (no split).

## Notes

- The unit tests are exactly the right shape for this kind of bug —
  they encode the previously-infinite inputs as finite-output
  expectations.
- `splitWordForWidth('@getskill.sh:3', 0).join('') === '@getskill.sh:3'`
  documents that zero-width inputs degrade to one-char-per-line rather
  than infinite-loop or data loss. Good behavior.
- `currentPart += char` on each grapheme is correct because `char`
  comes from `toCodePoints(word)`, so combining marks travel with their
  base. No surrogate-pair concerns.

Nothing actionable. Merge.
