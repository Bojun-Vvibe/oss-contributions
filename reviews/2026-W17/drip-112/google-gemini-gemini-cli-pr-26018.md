# google-gemini/gemini-cli PR #26018 — docs(cli): add skill discovery troubleshooting checklist to tutorial

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26018
- **Author**: @pmenic
- **Head SHA**: `5a441ef00306df0e365b832ee52def7e560af9e1`
- **Size**: +26 / −0
- **Files**: `docs/cli/tutorials/skills-getting-started.md`

## Summary

Pure docs PR. Adds a 5-item "If your skill doesn't appear" checklist to the skills-getting-started tutorial. The five items cover: (1) workspace-trust requirement for `<workspace>/.gemini/skills/`; (2) the two valid path layouts (`SKILL.md` at root vs one directory deep); (3) case-sensitive filename `SKILL.md` exactly; (4) frontmatter must be first-thing-in-file with `name:` and `description:`; (5) the displayed skill name comes from the `name:` field, not the directory name, and `: \ / < > * ? " |` are sanitized to `-`.

## Verdict: `merge-as-is`

Five real failure modes, each with a precise enough description that a user can self-diagnose without filing an issue. The "Files nested more than one directory deep are not discovered" call-out at item (2) and the "frontmatter must be the first thing — even a blank line breaks discovery" at item (4) are the kind of non-obvious gotchas that account for the long tail of "skill not loading" support questions.

## Specific references

- `docs/cli/tutorials/skills-getting-started.md:84-110` — the entire new section. Slotted right after "You should see `api-auditor` in the list of available skills." and before "## How to use the skill", which is the natural reading order: "I followed the tutorial, my skill didn't appear, what now?".
- Item 1, lines 91-94 — `<workspace>/.gemini/skills/` requires `/trust`, but `~/.gemini/skills/` (user scope) does not. This is the right level of asymmetry to call out — it's the single most common skill-discovery question because the security model is invisible until it bites.
- Item 2, lines 95-99 — "either at the root of the skills directory (`.gemini/skills/SKILL.md`) or one directory deep (`.gemini/skills/<skill-name>/SKILL.md`)... Files nested more than one directory deep are not discovered." This is exactly the kind of invariant that's worth pinning in docs because it's a runtime-silent failure in the skill-loader and doesn't surface anywhere in the UI.
- Item 3, lines 100-102 — `SKILL.md` exactly, case-sensitive. The "Linux, and macOS when configured as such" qualifier is precise; default macOS HFS+ is case-insensitive but APFS-on-newer-Macs and case-sensitive volumes are not. A `Skill.md` will work on default macOS and silently fail on Linux/CI.
- Item 4, lines 103-107 — frontmatter must be first-thing, no leading blank line, no H1 title before `---`. This is the gotcha that bites authors who copy from a markdown editor that auto-inserts a YAML preamble below an H1. Worth its own line.
- Item 5, lines 108-110 — name comes from `name:` field, with `: \ / < > * ? " |` sanitized to `-`. The sanitization list is suspiciously close to the Windows-illegal-filename character set (`< > : " / \ | ? *`), suggesting the skill loader uses the name as a filesystem path internally. That's a useful inference for an advanced reader and probably worth a separate note in a "skill name conventions" section.

## Nits

None blocking. Two follow-ups for a later PR:
1. Item 4 says "silently skipped if either field is missing" — would be nicer if the skill loader emitted a `--debug`-level log line ("skipped /path/to/SKILL.md: missing `description:` field"). That's a code change, not a docs change, but worth filing.
2. Consider adding item 6: "If multiple skills resolve to the same `name`, only one wins — which one is unspecified." This is a less common gotcha but it's the next-most-likely support question.

## What I learned

Documentation PRs that enumerate runtime-silent failure modes are disproportionately valuable because the failure modes are exactly the things the user can't discover from the tool itself. The skill-loader pattern here — `SKILL.md` at root *or* one-deep, frontmatter-first, case-sensitive filename, sanitized name field — has five separate "this fails silently" branches, and exposing them in the tutorial does more for adoption than any in-product error message would. The next iteration should be loud-failure logs (`--debug` skip-reason), but until then, the docs are the contract.
