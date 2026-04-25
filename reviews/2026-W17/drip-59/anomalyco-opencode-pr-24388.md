# anomalyco/opencode #24388 — docs: add opencode-local-ollama to ecosystem plugins

- URL: https://github.com/anomalyco/opencode/pull/24388
- Head SHA: `b54462c8171509db302638b58172a08b6017b367`
- State: OPEN
- Files: `packages/web/src/content/docs/ecosystem.mdx` (+1/-0)
- Closes: #24389
- Verdict: **merge-as-is**

## What it does

Single-row insertion in
`packages/web/src/content/docs/ecosystem.mdx:22`:

```
| [opencode-local-ollama](https://github.com/khalilgharbaoui/opencode-local-ollama) | Auto-discover local Ollama models and register them as native OpenCode providers |
```

The plugin auto-discovers running local Ollama models and registers
them as first-class OpenCode providers (no manual provider config).
Linked to npm: `opencode-local-ollama`.

## Diff notes

Inserted between `opencode-helicone-session` and `opencode-type-inject`.
Column widths and pipe alignment match the surrounding rows. Closes a
specific tracked issue (#24389) which is the right pattern for
ecosystem additions — gives reviewers a place to hash out
"is this canonical or is there already an `ollama-*` plugin we
prefer".

## Risk surface

- Pure docs change, no build-time validation involved (the
  ecosystem.mdx renders at static build time and broken links are
  caught by the existing link-check workflow if one runs over this
  file).
- Sibling PR #24390 from the same author adds
  `opencode-claude-code-plugin`. Both are reasonable ecosystem
  additions; reviewer may want to land them together for a single
  table churn.

## Why this verdict

One-line docs PR, properly issue-linked, alignment correct, plugin
purpose is non-controversial (every existing OpenCode auto-discovery
plugin sits in this same table). Lands clean.
