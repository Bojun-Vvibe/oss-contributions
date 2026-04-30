# sst/opencode PR #25087 — docs: add README in Indonesian

- **URL:** https://github.com/sst/opencode/pull/25087
- **Head SHA:** `7851571f9c90b2eee2de3f0040713a7e44d346a8`
- **Diff size:** +141 / -0 (single new file `README.id.md`)
- **Verdict:** `merge-after-nits`

## Specific diff references

- `README.id.md:1-48` — new top-of-file localized header that mirrors the existing English README structure (badges, language menu, screenshot link). The language switcher block at lines 18-41 lists all sister translations but does **not** add a link back to `README.id.md` itself; for consistency the other localized READMEs should be updated in the same PR (or a follow-up) to add `<a href="README.id.md">Bahasa Indonesia</a>` to their menus, otherwise this page becomes the only one that knows it exists.
- `README.id.md:49-95` — installation block translated to Bahasa Indonesia, with package-manager comments converted (`# OS apa pun`, `# atau bun/pnpm/yarn`). Command lines themselves are byte-identical to the canonical README, which is the right call — translating a `brew install` invocation would be a footgun.
- `README.id.md:104-141` — Agent / subagent section translates the operational text (`build`, `plan`, `general`) but keeps the reserved agent identifiers in English, matching how `README.zh.md` and `README.ko.md` handle the same paragraph. Worth a quick spot-check that the `Tab` key reference and the `@general` mention render correctly under Indonesian punctuation rules (the current copy at line 124 reads naturally).

## Reasoning

Pure documentation contribution with zero code/runtime impact, so the review bar is correctness of the translation and consistency with the rest of the localized README family. The diff is well-scoped: one new file, no edits to other READMEs, no asset churn. Translation quality on a spot-read is good — terms like "agen", "izin", and "subagent" are used consistently and the install commands are preserved verbatim, which avoids the classic "translated `sudo`" failure mode.

The single nit that pushes this off `merge-as-is`: the language-menu pattern in this repo is bidirectional. Every existing localized README has a list of sibling languages, and shipping `README.id.md` without adding it to the others' menus means Indonesian readers land on the page only via a direct link or search, not via the language switcher in any other locale. The fix is mechanical (one `<a>` line per file) but matters for discoverability, and PRs adding a new locale historically include the cross-links in the same change.

A second tiny consistency check worth doing before merge: the badge URLs at lines 12-14 reference `actions/workflows/publish.yml` on the `dev` branch — same as the upstream English README — so no drift there. The contributor explicitly notes "no functional or code changes," and the diff confirms that, so CI risk is nil. Approve once the sibling READMEs gain the Indonesian link.
