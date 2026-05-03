# block/goose #8979 — Improve readability in AGENTS.md

- **PR**: https://github.com/block/goose/pull/8979
- **Head SHA**: `3faeabb1de18121caef7e422639caf9075291532`
- **Author**: angiejones
- **Size**: +34 / −39, 1 file (`AGENTS.md` only)

## Files changed

- `AGENTS.md` (+34 / −39)

## Summary of change

Pure markdown reformat of `AGENTS.md`. The diff converts the
"Rules", "Code Quality", "Ink / Terminal UI (ui/text)", and "Never"
sections from bare-line `Key: value` entries into actual markdown
bulleted lists, and rewraps the long paragraphs in the Ink section so
each guideline is a single bullet rather than a hard-wrapped multiline
chunk. No semantic content changes.

## Specific-line review

- `AGENTS.md:78-83` — `Test:`, `Error:`, `Provider:`, `MCP:`, `Server:`
  bullets are now prefixed with `- `. This is the right call: most
  markdown renderers (GitHub web, IDE previews, mdbook) treat bare
  `Key: value` lines as a single soft-wrapped paragraph, which is
  exactly what made the original hard to scan.
- `AGENTS.md:88-95` (Code Quality) — same treatment: each
  `Comments:`, `Simplicity:`, `Errors:`, `Logging:` rule is now a top-
  level bullet. Content is byte-identical to the previous version.
- `AGENTS.md:100-115` (Ink section) — the most impactful chunk. The
  previous version hard-wrapped at ~95 cols with leading whitespace,
  which renders as awkward inline breaks on GitHub. The new version
  collapses each guideline to a single line per bullet. Note one
  inconsistency at lines ~104-106: `Ink-Layout` still has a hand-wrap
  ("Account for borders... computing the\nremaining space for dynamic
  text.") that mirrors the old format. Worth normalizing for
  consistency with the surrounding bullets.
- `AGENTS.md:117+` (Never section) — same reformat pattern. No content
  loss.

## Risk

Zero runtime impact. The only consumers that care are:
1. Humans reading the doc (improvement)
2. AI agents that load `AGENTS.md` into context (no semantic change,
   bullet structure is if anything easier to chunk)

## Things to consider (non-blocking nits)

- The leftover hand-wrap in the `Ink-Layout` bullet (noted above) is
  the only inconsistency. One-line fix.
- Some bullets have a trailing blank line, others don't. Tightening to
  uniform spacing would be a 30-second polish pass.

## Verdict

**merge-after-nits** — strictly improves readability with no semantic
loss, but the one stray hand-wrap in `Ink-Layout` should be normalized
before merge to leave the document fully consistent.
