# sst/opencode#25111 — docs: add 'aghast' to ecosystem

- **PR**: https://github.com/sst/opencode/pull/25111
- **Head SHA**: `8b6daac988fed577a4643667b44ac305a1517c15`
- **Verdict**: `merge-as-is`

## What it does

One-line addition to `packages/web/src/content/docs/ecosystem.mdx:55`,
adding `aghast` (https://github.com/BounceSecurity/aghast) — a security
code-scanning tool that uses the opencode SDK to navigate codebases — to
the third-party ecosystem table. +1 / -0.

## What's load-bearing

- `packages/web/src/content/docs/ecosystem.mdx:55` — single new table row
  matching the format of the surrounding entries (`opencode-worktree`,
  `opencode-sentry-monitor`, `opencode-firecrawl`). Column alignment is
  preserved, link target is reachable, description is one sentence and
  truthful about the tool's purpose ("Secure code scanning with LLMs which
  uses the OpenCode SDK to navigate the codebase").
- The PR was opened in response to the page's own "If you'd like to add
  your project, open a PR" invitation per the PR body. Author is the
  project's maintainer (Bounce Security), so attribution is genuine, not
  drive-by spam.

## Concerns

None that block. The repo (`BounceSecurity/aghast`) is a security tool but
this PR doesn't *contain* security samples — it's a single link in an
ecosystem table, structurally identical to the surrounding entries. The
project itself is an LLM-driven SAST tool (analysis, not payloads), which
is the type of security tooling the ecosystem page already lists
(`opencode-sentry-monitor` for observability, etc.).

## Nits (none required)

- The description column already has wider entries; could be tightened to
  match the average length, but the current text is more informative than
  several existing rows. Not worth a round-trip.

## Verdict

`merge-as-is` — single-line, format-conformant, author-attributable docs
addition responding to an open invitation on the page itself.
