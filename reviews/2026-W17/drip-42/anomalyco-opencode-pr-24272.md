# anomalyco/opencode PR #24272 — docs: add Mongolian README documentation

- **URL:** https://github.com/anomalyco/opencode/pull/24272
- **Head SHA:** `d4aa1133f1a5c93242f1ffbe5599024c1a123c5f`
- **Files touched:** 23 (1 new `README.mn.md` + 22 existing `README.*.md` patched to add a link to the Mongolian translation)
- **Verdict:** `merge-after-nits`

## Summary

Adds a new Mongolian-language `README.mn.md` (142 lines) and
back-links it from every other locale README's language switcher, so
the new translation shows up in the menu regardless of which README
the reader lands on.

## Specific references

- `README.mn.md` (new, +142) — full translation, follows the same
  section structure as `README.md` (Install / Documentation / FAQ /
  Contributing).
- `README.md` and 21 sibling locale READMEs — each gets `+2 / -1`,
  inserting the `[Монгол](README.mn.md)` link into the language list
  in the same alphabetical slot. The uniform diff confirms the author
  used a single edit pattern, not hand-edits.
- No code, schema, codegen, or generated SDK files touched — pure
  docs.

## Reasoning

Translation PRs are low-risk and high-leverage when the author
remembers to update the language switcher in every other locale, and
this one does — the per-file diff is mechanically uniform across all
22 sibling READMEs.

Two nits worth raising before merge:

1. Verify the Mongolian rendering uses Cyrillic Mongolian throughout
   and not a mix of Cyrillic + traditional Mongolian script —
   reviewers can't readily catch this and it'll be a maintenance
   burden later.
2. The author should confirm there's no broken anchor in the
   translated section headings (the original `README.md` uses
   English-anchored fragment links in a couple of places).

Otherwise good to merge — no blocker.
