# Review: block/goose #8974 — Add VC Deal Flow Signal MCP to extensions directory

- **Repo**: block/goose
- **PR**: #8974
- **Head SHA**: `ee5e4e1d08f7db4cdd7b87fe29b7aee82ea2382c`
- **Author**: kindrat86

## What it does

Single-file change to `documentation/static/servers.json`: appends a
new MCP server entry (`vc-deal-flow-signal`) to the public extensions
directory. Six read-only tools backed by `npx -y @gitdealflow/mcp-signal`,
no auth, marked `endorsed: false` (community submission).

## Diff notes

- `documentation/static/servers.json:856-867` — well-formed JSON entry
  matching the existing `npx-stdio` schema (mirrors the AgentQL /
  Tavily entries cited in the PR body). Required fields present:
  `id`, `name`, `description`, `command`, `link`, `installation_notes`,
  `is_builtin: false`, `endorsed: false`, `environmentVariables: []`.
- `documentation/static/servers.json:39` and `:851` — two unrelated
  diff hunks: the JSON file was re-emitted with `\u26a1` and `\u2014`
  in place of the literal `⚡` and `—` in two existing entries (Apify
  and Cash App). This is a serializer artifact, not an intentional
  change. The rendered string is identical, but reviewers should
  confirm the project's lint/format step accepts both representations
  — if the upstream tooling normalizes to literal Unicode, this could
  trip a CI check or cause a roundtrip on the next edit.

## Concerns

1. **Author has no prior contributions to this repo** (per the PR
   metadata). The directory is a low-risk surface (just a JSON entry,
   no code execution at directory-build time), but maintainers should
   sanity-check that `@gitdealflow/mcp-signal` on npm actually
   implements the six tools claimed in the description before
   endorsing the entry as a discoverable extension. The
   `endorsed: false` flag mitigates the risk.
2. The two collateral encoding changes (lines 39, 851) should be
   reverted to keep the diff minimal — they make code review noisier
   and could conflict with an upstream lint pass. Ask the author to
   re-run their formatter with the same Unicode policy as the existing
   file, or hand-revert those two hunks.
3. The `link` points to `github.com/kindrat86/mcp-deal-flow-signal`
   (the source repo), not to a public-facing landing page. Consistent
   with most other entries in the file, but a few entries (e.g.
   Cash App) leave `link` empty when there's a streamable-http URL.
   Non-blocking.
4. No license/security review at the directory-listing layer — that's
   the project's existing posture for `endorsed: false` entries, so
   not a regression.

## Verdict

merge-after-nits
