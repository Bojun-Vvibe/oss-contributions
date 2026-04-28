# sst/opencode #24746 — docs: add opencode-working-memory to ecosystem

- PR: https://github.com/sst/opencode/pull/24746
- Author: sdwolf4103
- Head SHA: `942a4cc0c7f1`
- Base: `dev` · State: OPEN
- Diff: 1+/0- across 1 file (`packages/web/src/content/docs/ecosystem.mdx`)

## Verdict: merge-after-nits

## Rationale

- **Single-row addition to the ecosystem table at `ecosystem.mdx:43`.** Drops `[opencode-working-memory](https://github.com/sdwolf4103/opencode-working-memory)` directly underneath the existing `opencode-supermemory` row — an intentional grouping by *what the plugin does* (persistent memory) rather than by name-sort. Reasonable curation choice; the surrounding rows show the file is loosely topic-clustered (memory plugins together, scheduling plugins together, etc.) so this placement matches the editorial pattern already in use.
- **Description is the right shape for a one-row plugin entry.** "Automatic memory for OpenCode agents across compactions and sessions" names two distinct contexts where state evaporates (compaction, session boundary) and frames the plugin as the patch — that's strictly more useful than the supermemory row above it ("Persistent memory across sessions") which leaves the compaction case unspecified. Author of this PR is also the author of the linked plugin (`sdwolf4103` matches the GitHub org), which is the standard self-submission pattern for ecosystem table additions.
- **Zero risk to the build.** The `.mdx` table is rendered statically; row insertion can't break sibling rows. Author's PR body confirms `bun typecheck` passes (table cells are plain text, no MDX components). The "How did you verify your code works?" checklist is filled in.

## Nits / follow-ups

- **Verify the linked repo exists and is non-malicious before merge.** The supply-chain risk of an ecosystem-listing PR is that anyone can submit a row pointing at a malicious package. The linked repo `sdwolf4103/opencode-working-memory` should be live, have a README that matches the description, declare an MIT/Apache license, and not pull in any obvious typosquats. Standard ecosystem-doc-PR triage; not a blocker if the repo checks out.
- **Description column width is the only readability concern.** The row uses ~95 characters of description (longer than most siblings) which may push the `npm`-version column on narrow screens. Easy to verify in the rendered docs preview before merge — if it wraps badly, trim "across compactions and sessions" → "across sessions" and let the README explain the compaction angle.
- **Manual table sort is fragile long-term.** The file is now ~40+ rows of hand-curated ordering. A future PR proposing alphabetical sort or topic-cluster headings would be a higher-leverage cleanup than this one — but that's out of scope here.
- **Author should add a one-line "What this plugin does differently" to their own README** if they haven't, since the ecosystem table only has room for ~15 words. Two persistent-memory plugins side-by-side (supermemory + working-memory) need a discoverability differentiator at the README level.

## What I learned

Ecosystem-listing PRs are a trust-distribution surface that looks like documentation but functions like a tiny package registry. The maintainer accepting a row is implicitly endorsing that *clicking the link is safe* — even though they have no leverage over what the linked repo ships next month. The right discipline is: (1) verify the repo exists and has a README that matches the row, (2) trust-but-don't-deeply-vet the contents (it's not a hard fork), (3) prefer descriptions that name the *gap the plugin fills* over generic "adds X" framing, because the next person browsing the table is shopping for a solution to a specific problem. This row hits (3) cleanly by naming "compactions and sessions" — the two specific places the agent's memory evaporates — rather than the vaguer "persistent memory" framing one row above.
