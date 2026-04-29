# sst/opencode #24941 — add Skills section to ecosystem page

- **PR:** https://github.com/sst/opencode/pull/24941
- **Title:** `docs: add Skills section to ecosystem page`
- **Author:** LDLZM
- **Head SHA:** 44ac080488ae9983698aa590b6f6d14b694f06c6
- **Files changed:** 1 (`packages/web/src/content/docs/ecosystem.mdx`),
  +13 / −0
- **Verdict:** `merge-after-nits`

## What it does

Adds a new "Skills" section to the ecosystem page, above the existing
"Plugins" section. Lists four community Skills with name, link, and
short description:

- `opencode-mcp-registry` — register new MCP servers
- `uv-skill` — use `uv` for Python package management
- `crowdsource-debugger` — debug via Stack Overflow / GitHub Issues / Reddit / Discourse
- `sensitive-operations` — delegate passwords/sudo/tokens/destructive commands to the user

Diff at `ecosystem.mdx:14-25` adds:

```mdx
## Skills

OpenCode [Agent Skills](/docs/skills/) are reusable instructions that
agents load on-demand. Install by cloning into `~/.claude/skills/`.

| Name | Description |
| ---- | ----------- |
| [opencode-mcp-registry](https://github.com/LDLZM/opencode-mcp-registry) | Register new MCP servers in opencode — install, configure auth, and test |
| [uv-skill](https://github.com/LDLZM/uv-skill) | Use uv for ALL Python package management — replaces pip/poetry/pyenv |
| [crowdsource-debugger](https://github.com/LDLZM/crowdsource-debugger) | Debug stubborn errors via Stack Overflow, GitHub Issues, Reddit, and Discourse |
| [sensitive-operations](https://github.com/LDLZM/sensitive-operations-skill) | Delegate passwords, sudo, tokens, and destructive commands to the user |

---
```

## What's good

- Placement is sensible: Skills sit above Plugins because they're the
  newer, lower-friction extension surface and most readers landing on
  the ecosystem page will be looking for them first.
- The intro line links to `/docs/skills/` and gives a one-sentence
  install instruction (`~/.claude/skills/`), which means a reader can
  grok the section without bouncing to another doc page.
- Table format matches the existing Plugins table immediately below it
  (same column count, same use of `[name](url)` link cells), so the
  visual rhythm of the page is preserved.

## Nits / risks

- All four Skills listed are authored by the same GitHub user (LDLZM —
  the PR author). That's fine for seeding a new section, but it would
  read better if the section opened with a one-liner like
  *"To submit your own Skill, open a PR adding a row below."* so the
  table doesn't look like a single author's portfolio. Consider also
  pointing at `awesome-opencode` (already linked at the top of the
  page) as the longer-tail discovery surface.
- The install instruction `~/.claude/skills/` is correct for users
  running OpenCode alongside Claude Code, but a reader who only uses
  OpenCode may be confused by the path. Either:
  - clarify ("Install by cloning into `~/.claude/skills/` (shared with
    Claude Code) or your OpenCode skills directory"), or
  - link to the install section of `/docs/skills/` and let that page
    own the path.
- Markdown table rows use trailing whitespace for column alignment
  (you can see padding with multiple spaces in the diff). MDX
  renderers handle this fine, but `prettier --write` may collapse
  them on the next pass through and produce a noisy diff. If the
  repo runs prettier in CI, formatting may already match; if not,
  worth a one-shot run before merge.
- The `sensitive-operations` row links to a repo named
  `sensitive-operations-skill` while the displayed name is
  `sensitive-operations` — minor inconsistency. Either rename the
  display or use the actual repo name in the visible label so users
  can copy-paste-grep.

## Verdict rationale

`merge-after-nits`: pure docs change with low blast radius, but the
"all entries by one author" + ambiguous install path are worth a quick
polish before merge so the section is clearly inviting follow-on
contributions and not confusing for OpenCode-only users.
