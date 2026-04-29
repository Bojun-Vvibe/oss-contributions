# sst/opencode#24971 — docs: add BRHP to ecosystem plugins

- **PR**: https://github.com/sst/opencode/pull/24971
- **Head SHA**: `d50ce8ea6d94c8a0a81f98d8567af043159a1c8d`
- **Size**: +1 / −0, 1 file
- **Files**: `packages/web/src/content/docs/ecosystem.mdx`
- **Refs**: #24970

## Context

A single new row inserted into the ecosystem-plugins markdown table at `packages/web/src/content/docs/ecosystem.mdx:47`, between the `opencode-conductor` row and the `micode` row, linking to https://github.com/ZanzyTHEbar/brhp with the description: "Structured, persistent planning with local session state, `/brhp` commands, and a TUI sidebar".

## Why this exists / problem solved

Per the PR body and the `Refs #24970` link, this is the docs side of an ecosystem-listing request — the author of BRHP ("Brain Rot Helper Plugin", per the linked repo name pattern) wants their plugin discoverable in the upstream ecosystem index. opencode's docs page is the canonical landing spot for third-party integrations, and the existing table is curated alphabetically-ish but actually appears to be insertion-order, so any new entry is just "find a reasonable home and add a row."

## Design analysis

The diff is mechanically a one-line table-row insertion at `ecosystem.mdx:47`. The new row matches the column shape of the surrounding rows (`| name + link | description |`) and the description is a reasonable one-line summary of what the plugin does — "structured persistent planning + `/brhp` commands + TUI sidebar" matches the actual feature set advertised on the linked repo.

Two minor things worth noting from reading the surrounding context:

1. **Insertion location.** The row lands between `opencode-conductor` (a "Context → Spec → Plan → Implement workflow" plugin) and `micode` (a "Brainstorm → Plan → Implement workflow with session continuity" plugin). BRHP, with its planning + session-state + commands theme, fits cleanly in this neighborhood — it's adjacent to other planning/session-management plugins. So the placement is sensible, not just "appended to the bottom".
2. **Column-alignment whitespace.** The new row uses slightly different padding than its neighbors — 99 visible characters in the description column vs. ~99 in surrounding rows — and the `|` column alignment isn't perfectly preserved (the inserted line has an extra leading space before `BRHP` to push past the `|` border). MDX/markdown table rendering doesn't care about exact column alignment, so this is purely a source-readability nit, but a `prettier` or `markdownlint` pass on the file would normalize it for diff cleanliness on the next round of additions.

What's missing from the PR is any verification that the linked repo (`https://github.com/ZanzyTHEbar/brhp`) actually exists, builds against current opencode, has a license, and isn't a name-squat or low-quality fork. The opencode ecosystem doc functions as a trust signal — appearing in the upstream's official docs lends legitimacy to a third-party plugin — so the maintainer should do a 30-second sanity check on the target repo (does it have any commits in the last 90 days? does it have a README? does it have a non-trivial code surface?). The PR body doesn't show this check was done.

## Risks

- Zero runtime/build risk — single docs-page row addition, MDX table parser handles it, nothing executes from this file.
- Trust/curation risk: linking to a third-party repo from official docs lends credibility. If the target repo is abandoned, broken, or malicious, future opencode users following the link could have a bad experience. This is a one-time review-cost, not an ongoing risk.
- Long-term: the ecosystem table is now ~25+ entries and growing without categorization. At ~50 entries it'll need either alphabetization, grouping by capability (planning / TUI / agents / dev-tools), or a search facet — but that's a follow-up for the docs maintainer, not blocking on this PR.

## Suggestions

1. Confirm the target repo `ZanzyTHEbar/brhp` exists, has a license file (so users can audit usage rights), and has at least a README that matches the description claim ("local session state, `/brhp` commands, TUI sidebar"). A 30-second visit to the link is sufficient.
2. Run `prettier --write packages/web/src/content/docs/ecosystem.mdx` (or whatever the project's markdown formatter is) before merge to normalize the column-alignment whitespace so the next PR adding a row gets a clean diff. This is a one-line fix-on-merge, not blocking.
3. Open a follow-up issue (or add a TODO comment to the file) noting that the ecosystem table is approaching the threshold where it'd benefit from categorization or a generated-from-frontmatter approach. The current "humans append rows" model scales linearly with PR cost; a frontmatter-driven catalog would amortize.
4. If maintainers want a higher quality bar for ecosystem entries, document the inclusion criteria in the file's intro (e.g., "Entries should link to active repos with a README and license; remove inactive entries quarterly"). This makes review of future PRs easier and gives gentle rejection language for low-quality submissions.

## Verdict

**merge-after-nits** — the row is well-formed and well-placed; the only real ask is a 30-second sanity check on the linked repo before merging (does it exist, does it have a license, is it more than a stub). Optional whitespace normalization is a fix-on-merge, not blocking.

## What I learned

Ecosystem-listing PRs are deceptively low-stakes — the diff is one line, the build risk is zero, and the docs render the same way regardless. But the *trust* the maintainer extends by accepting the listing is non-trivial: an entry in opencode's official ecosystem docs is a strong "we acknowledge this exists" signal that downstream users read as endorsement. The right reviewer policy for this kind of PR is "fast-merge once the target repo passes a 30-second smell test (exists, has README, has license, has activity)" rather than "fast-merge unconditionally" — and that distinction is worth writing into a CONTRIBUTING note so PR authors know to include the smell-test evidence in their PR body to speed merge.
