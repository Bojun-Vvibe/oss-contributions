# oss-contributions

A public log of my open-source contributions and reviews.

This repository collects two kinds of artifacts:

1. **Drafted PR reviews** — careful, long-form reviews of open pull requests in
   public OSS projects I follow. Each review walks through the diff,
   surfaces risks, asks the questions a thoughtful maintainer would ask,
   and records what I learned from reading it.
2. **Cross-cutting insights** — periodic syntheses (`reviews/INSIGHTS.md`)
   that extract themes across many reviews — what's actually changing in
   the AI coding-agent ecosystem week to week.

## Important disclaimer

**Drafted reviews in this repo are NOT automatically posted to upstream
projects.** They are public reading material — a study log. If I decide
to engage upstream on a particular PR, I do so manually, with a focused
comment, after re-reading and trimming. Treat everything under
`reviews/` as my notes, not as a maintainer's verdict on the PR.

## Layout

```
reviews/
  <repo-flat>/        # e.g. anomalyco-opencode/, BerriAI-litellm/
    PR-<number>.md
  INDEX.md            # all reviews, organized by repo
  INSIGHTS.md         # cross-PR theme extraction
```

Each review file uses YAML frontmatter and a fixed seven-section structure
(Context, Diff walkthrough, Risks, Questions, Verdict, What I learned).

## Targets I'm tracking

The current rotation of repos I sample PRs from:

- [anomalyco/opencode](https://github.com/anomalyco/opencode) — terminal coding agent
- [BerriAI/litellm](https://github.com/BerriAI/litellm) — model gateway / proxy
- [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) — reference MCP servers
- [openai/codex](https://github.com/openai/codex) — agentic CLI
- [OpenHands/OpenHands](https://github.com/OpenHands/OpenHands) — autonomous coding agents
- [cline/cline](https://github.com/cline/cline) — VS Code coding agent
- [Aider-AI/aider](https://github.com/Aider-AI/aider) — pair-programming CLI
- [charmbracelet/crush](https://github.com/charmbracelet/crush) — terminal coding agent

## License

- Code snippets and scripts: **MIT** (see `LICENSE`).
- Prose (review write-ups, insights, synthesis): **CC-BY-4.0**.

Quoted diff snippets remain under the upstream project's license; they are
included here under fair-use review/commentary, with attribution to the
source PR via the URL in each file's frontmatter.

## Index

See [`reviews/INDEX.md`](reviews/INDEX.md) for the full list, and
[`reviews/INSIGHTS.md`](reviews/INSIGHTS.md) for cross-PR themes.
