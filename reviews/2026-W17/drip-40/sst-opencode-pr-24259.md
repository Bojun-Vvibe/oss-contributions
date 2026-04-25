# sst/opencode #24259 — docs: add opencode-simple-notify to ecosystem

- **Repo**: sst/opencode
- **PR**: [#24259](https://github.com/sst/opencode/pull/24259)
- **Head SHA**: `404a50fc1009edef0e4d5f5cefa606487341fc06`
- **Author**: Yusuzhan
- **State**: OPEN (+1 / -0)
- **Verdict**: `merge-as-is`

## Context

Single-line addition to the ecosystem doc page listing
third-party plugins. The PR adds
`opencode-simple-notify` (a desktop-notification plugin for
Linux/Mac) to the table of community plugins on the
ecosystem page.

## Design

`packages/web/src/content/docs/ecosystem.mdx:55` — one row
appended to the existing markdown table:

```mdx
+| [opencode-simple-notify](https://github.com/Yusuzhan/opencode-simple-notify) | Lightweight desktop notifications for OpenCode (Linux, Mac) |
```

Inserted in source order (after `opencode-firecrawl`, before
the `---` separator), which matches the convention used by
the other entries — the table is not alphabetical, just
chronological-ish by submission. The link target is HTTPS,
the description fits the column width, and the trailing pipe
syntax matches the surrounding rows exactly.

## Risks

- **Self-promotion verification.** Ecosystem listings are
  the cheapest possible vector for "shipping" a plugin that
  doesn't actually work or that does something hostile (data
  exfiltration via "notification content"). A reviewer
  should at minimum:
  1. Open the linked repo, confirm it has a real
     `package.json` with a `plugin` export shape that matches
     the host's plugin contract.
  2. Skim the source for obvious red flags (network calls
     beyond OS notification APIs, fs writes outside the
     plugin's own state dir, `eval` / dynamic `require`).
  3. Check the licence — community plugins usually need to
     be MIT/Apache-2.0/ISC for the listing to be safe to
     promote.
  None of this needs to be in the PR; it's the maintainer's
  call. But "+1 line, docs only" is exactly when supply-chain
  hygiene tends to lapse.
- **Description claim ("Linux, Mac")** — no Windows. If the
  ecosystem page elsewhere advertises Windows support as a
  baseline, the omission is worth calling out so users on
  Windows don't try to install and hit a runtime error. Worth
  a one-word clarification: `(Linux, Mac only)`.
- **Table column-width drift.** The pipe-table's whitespace
  alignment in the existing rows is consistent (the
  description column starts at column ~100). The new row
  appears to match — but on a `+1 / -0` diff it's hard to
  confirm pipe-position without re-running the markdown
  formatter the project uses (probably `prettier --parser
  markdown` given the rest of the codebase). If a doc-format
  CI step exists, it'll catch any misalignment.

## Suggestions

- (Nit) Tighten the description: `Lightweight desktop
  notifications (Linux, Mac only)` — drops the redundant
  "for [project]" since context is already the
  ecosystem-of-[project] page.
- Maintainer should do a quick supply-chain glance at the
  linked repo (see Risks #1) before merging. Routine for new
  ecosystem entries.

## Verdict reasoning

`merge-as-is`. It's a one-line community plugin listing
following the established convention. The supply-chain
hygiene check is on the maintainer regardless of how the PR
is structured, and the description-clarity nit is taste.

## What I learned

Ecosystem / community-plugin doc pages are the lowest-friction
PRs for an open-source project — and consequently the
highest-leverage attack surface for a typosquat or hostile
plugin. The mitigation isn't "make the PR harder to land,"
it's "have a checklist for the reviewer." A
`docs/ECOSYSTEM_REVIEW_CHECKLIST.md` (linked from the PR
template for changes touching `ecosystem.mdx`) would make
the supply-chain hygiene step explicit without making
contributors jump through hoops. Items: (1) repo has a real
`package.json`, (2) plugin export shape matches the host's
contract, (3) licence is permissive, (4) no obvious data
exfiltration, (5) maintainer is identifiable (not a
two-week-old throwaway account). That's a 5-minute review
that closes most of the risk.
