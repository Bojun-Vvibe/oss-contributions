# Review — sst/opencode#25281

- PR: https://github.com/sst/opencode/pull/25281
- Title: fix(tui): long tips truncated in TUI home view
- Head SHA: `d47fd19dabd3599e7001aacd50012c6b57682dba`
- Size: +19 / −6 across 2 files
- Verdict: **merge-after-nits**

## Summary

Three tips in the home view exceeded the visible width (75 cols minus the
"● Tip " prefix → 69 chars) and were getting truncated mid-word. The PR
shortens the offending strings, exports the `TIPS` array, and adds a unit
test that fails fast if any future tip blows past the 69-char budget after
stripping `{highlight}…{/highlight}` tags.

## Evidence

- `packages/opencode/src/cli/cmd/tui/feature-plugins/home/tips-view.tsx`:
  the `const TIPS` becomes `export const TIPS`, with a comment explaining
  the 69-char cap. Five tips trimmed (e.g. the `/share`, the docker, the
  `scroll_acceleration`, env-var, and `opencode.json` tips).
- `packages/opencode/test/cli/tui/tips.test.ts` (new): filters
  `TIPS` for entries whose stripped length > 69 and asserts the
  result is `[]`.

## Notes / nits

- The width cap (`69`) and the parent's width (`75`) are both magic
  numbers that live in different files. Worth a follow-up to import a
  shared constant (e.g. `TIP_PREFIX = "● Tip "` and `TIPS_MAX_WIDTH =
  75`) so a parent-width change can't silently regress this test.
- `replace(/\{\/?highlight\}/g, "")` is duplicated between the test and
  whatever renders tips at runtime. Extracting a `stripHighlights()` helper
  would prevent test/runtime drift if more inline tags ever appear.
- The trimmed copies are mostly fine, but "Use docker run --rm
  ghcr.io/anomalyco/opencode" drops the `-it` flag — `-it` is required
  for an interactive container. Consider `docker run -it --rm ghcr.io/…`
  or rephrase to "containerized usage" without an exact command.

Otherwise straightforward; ship after the `-it` nit.
