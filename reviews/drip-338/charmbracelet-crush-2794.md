# Review: charmbracelet/crush #2794

- **Title:** chore(deps): bump github/codeql-action from 4.35.2 to 4.35.3 in the all group
- **Head SHA:** `ccd37a5bc1bf68ab7aaf533ea69fd036f6296efc`
- **Scope:** +5 / -5 single file `.github/workflows/security.yml`
- **Drip:** drip-338

## What changed

Dependabot bump of `github/codeql-action` from `v4.35.2` (SHA-pinned
`95e58e9a2cdfd71adc6e0353d5c52f41a045d225`) to `v4.35.3` (SHA-pinned
`e46ed2cbd01164d986452f91f178727624ae40d7`). All four references in the
security workflow (`init`, `autobuild`, `analyze`, two `upload-sarif`)
are updated in lockstep.

## Specific observations

- `.github/workflows/security.yml:33,35,36` — `init`, `autobuild`,
  `analyze` all pinned to the new `e46ed2cb…` SHA with the `# v4.35.3`
  trailing comment preserved. Good convention — keeps the action
  immutable while leaving the human-readable version visible.
- `.github/workflows/security.yml:55` — `upload-sarif` after the Grype
  scan also bumped consistently. Easy to miss this one in mixed-action
  workflows; the dependabot grouping caught it.
- `.github/workflows/security.yml:76` — second `upload-sarif` after the
  govulncheck step also updated. All four CodeQL action references are
  on the same revision now, no stale pin left behind.
- The bump is a patch release within the v4 major. CodeQL action 4.35.x
  has been stable; no breaking changes expected. Cross-reference release
  notes to confirm no new required inputs were introduced for the
  `analyze` step.
- All dependent action references live in this single `security.yml`
  file — no other workflow under `.github/workflows/` calls
  `github/codeql-action`. The dependabot grouping covered the full
  blast radius.

## Risks

- Patch-level bump of a CodeQL release is low-risk by GitHub's published
  semver contract for actions; downside is bounded to a CI failure that
  surfaces immediately on next push.

## Verdict

**merge-as-is** — clean, fully-pinned, dependabot-shaped patch bump with
no leftover stale references.
