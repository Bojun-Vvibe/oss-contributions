# anomalyco/opencode #24390 — docs: add opencode-claude-code-plugin to ecosystem

- URL: https://github.com/anomalyco/opencode/pull/24390
- Head SHA: `e3e16550285af847d7b9c2b231153cc3da45e134`
- State: OPEN
- Files: `packages/web/src/content/docs/ecosystem.mdx` (+1/-0)
- Verdict: **merge-after-nits**

## What it does

Single-row addition to the ecosystem plugin list at
`packages/web/src/content/docs/ecosystem.mdx:55`, pointing at
`khalilgharbaoui/opencode-claude-code-plugin`. The plugin wraps the
Claude Code CLI and routes model traffic through it (Pro/Max,
Bedrock, Vertex, API-key) instead of the Anthropic HTTP API.

## Diff notes

The single new row:

```
| [opencode-claude-code-plugin](https://github.com/khalilgharbaoui/opencode-claude-code-plugin) | Use your Claude Code CLI auth (Pro/Max, Bedrock, Vertex, API key) with opencode's UI, agents, and MCP |
```

Inserted just after `opencode-firecrawl` at the bottom of the table.
Column alignment matches surrounding rows.

## Risk surface / nits

- **No de-dup check vs. similar entries.** The same author appears to
  have a sibling PR #24388 adding `opencode-local-ollama`. Both come
  from the same fork (`khalilgharbaoui/*`). That's fine for ecosystem
  pages, but worth a maintainer eyeball that the upstream repo is
  active and not e.g. a fork-and-rename of an older deprecated
  plugin.
- **Trust review for the linked tool.** The ecosystem table is
  effectively a "softly endorsed" surface — users skim it for what
  to install. The linked plugin shells out to Claude Code CLI, which
  by extension brings Claude Code's auth surface into the OpenCode
  process. Not a reason to reject (other rows in the table have
  similar shapes — `opencode-codex-auth`, `opencode-gemini-auth`),
  but the ecosystem page should ideally have a one-liner header note
  about "third-party plugins, audit before installing". That belongs
  in a follow-up PR, not this one.
- **Ordering.** Other ecosystem tables in this file appear to be
  loosely topical/alphabetical; the new entry lands at the bottom,
  matching how the recent rows have been added. Fine as-is.

## Why this verdict

Pure docs, single line, no code paths. The "nits" above are about
the ecosystem page as a whole, not this PR's correctness — happy
to merge after the maintainer confirms the linked plugin is
maintained and the row sort order matches their preference.
