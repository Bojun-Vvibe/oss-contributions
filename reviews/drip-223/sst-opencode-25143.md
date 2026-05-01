# sst/opencode#25143 — docs: Add opencode-swarm to ecosystem documentation

- Repo: sst/opencode
- PR: https://github.com/sst/opencode/pull/25143
- Head SHA: `9e15d5e35d77`
- Size: +1 / -0 (one file)
- Verdict: **merge-as-is**

## What the diff actually does

One line, one file: `packages/web/src/content/docs/ecosystem.mdx`, line 55.

Inserts a single Markdown table row immediately after the
`opencode-firecrawl` row in the Plugins table:

```
| [opencode-swarm](https://github.com/zaxbysauce/opencode-swarm) | Verification-gated swarm with architect, review, test, and security agents and resumable evidence |
```

## Why merge-as-is

- **Format-preserving insertion.** Pipe count, column padding, and link
  shape match the surrounding rows exactly (compare with
  `opencode-firecrawl` and `opencode-sentry-monitor` directly above). MDX
  table parsers are whitespace-tolerant but a future column-realignment
  pass would re-pad anyway.
- **Description is concrete and short.** "Verification-gated swarm with
  architect, review, test, and security agents and resumable evidence" is
  one clause naming the four agent types and the differentiator (resumable
  evidence) — same density as the neighboring rows ("Web scraping, crawling,
  and search via the Firecrawl CLI").
- **Pure ecosystem listing.** No code, no schema, no build step, no risk
  surface. The merge bar for "add a row to the ecosystem table" is
  intentionally low — it's the docs equivalent of an awesome-list addition.

## Nits I would NOT block on

- The neighbour-row pattern uses sentence-fragment descriptions
  (capitalized first word, no trailing period) and the new row matches —
  good. A picky reviewer might want the trailing "evidence" clarified
  ("resumable verification evidence" vs "resumable agent state") but the
  upstream README at the linked repo can carry that nuance.
- No screenshot or "did you verify the rendered page" note in the PR body,
  but for a one-row table addition the diff is self-evidencing.

## Theme tie-in

Spec-form contribution at the lowest possible scope — one row in a table
that is itself the canonical spec for "what plugins exist." The
contribution mechanism (open a PR, add a row) is the right operator
boundary for "I built a thing, here's where it lives in the registry."
