# google-gemini/gemini-cli #26418 — Docs audit: 2026-05-04

- Link: https://github.com/google-gemini/gemini-cli/pull/26418
- Head SHA: `e8c014a55536217fed955f4670c743b699d05cda`
- State: OPEN, +119/-6 across 7 files

## Summary
A scheduled docs audit that (a) ships two transcript artifacts
(`audit-implementation-log-2026-05-04.md`, `audit-results-2026-05-04.md`)
documenting the audit decisions, and (b) lands the actual doc edits:
adds missing introductory overview paragraphs to several `## ` headings
in `docs/cli/cli-reference.md`, replaces "Click on **Sign in**" with
"Click **Sign in**", removes "We recommend" voice in
`docs/get-started/installation.mdx`, and adds a "Next steps" section.
Also fixes a minor heading reference in `docs/index.md`.

## Specific references
- `audit-results-2026-05-04.md` — Phase-1 audit findings table
  enumerating each violation/inaccuracy and the recommended fix. Lists
  `/gemma` command and "Advanced UI Settings" as undocumented features.
- `audit-implementation-log-2026-05-04.md` — decisions table mapping each
  finding to Accept / Reject / Modify with reasoning. Includes a
  Conseca-related "future improvement" follow-up.
- `docs/cli/cli-reference.md:8-10` — adds overview before
  `## CLI commands` table.
- `docs/cli/cli-reference.md:49-51, 96-98, 118-120, 135-137` — same
  pattern for `## CLI Options`, `## Extensions management`,
  `## MCP server management`, `## Skills management`.
- `docs/cli/cli-reference.md:152-156` — adds a "Next steps" section
  linking to commands reference and settings.
- `docs/get-started/index.md:46` — "Click on **Sign in**." → "Click **Sign in**.".
- `docs/get-started/installation.mdx:25-26` — removes "We recommend"
  voice; replaces with "Most users install …".
- `docs/get-started/installation.mdx:76` — "we recommend running" →
  "Use the `gemini` command to start an interactive session:".
- `docs/get-started/installation.mdx:160-161` — same for the Releases
  section ("we recommend the stable" → "Most users use the stable").

## Notes
- Net positive: the audit log/results files are useful provenance and
  they shipped at the same time as the actual edits. Many docs PRs
  ship one without the other.
- One small concern: shipping `audit-implementation-log-*.md` and
  `audit-results-*.md` at the **repo root** clutters the top-level tree.
  Consider moving them to `docs/audits/` or a similar subdirectory so
  successive audits don't accumulate at root level.
- Voice fixes are uniformly good — replacing "we recommend" with
  imperative ("Use the `gemini` command") is more direct and matches the
  style guide cited in the audit log.
- Audit log notes one remaining unchecked box ("Run `npm run format`")
  — reviewer should confirm formatting was applied before merge.

## Verdict
`merge-after-nits` — relocate the two top-level audit `.md` files into
`docs/audits/` (or the agreed-on subdir) and confirm `npm run format`
was run.
