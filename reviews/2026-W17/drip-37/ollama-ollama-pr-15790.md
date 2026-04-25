# ollama/ollama #15790 — Add Atlarix to Code Editors & Development integrations

- **Repo**: ollama/ollama
- **PR**: [#15790](https://github.com/ollama/ollama/pull/15790)
- **Head SHA**: `1e6c8bc0066e8fb79c6bb1ea8a5593fea5f388ed`
- **Author**: AmariahAK
- **State**: OPEN (+1 / -0)
- **Verdict**: `merge-after-nits`

## Context

README.md "Code Editors & Development" integrations list addition:
one new bullet for Atlarix, a native macOS/Linux desktop AI coding
environment that uses Ollama as a first-class local provider. PR
description claims a "Round-Trip Engineering" Blueprint graph
(SQLite) for RAG, reducing per-query token usage from ~100K to
~5K.

## Design

Single-line addition at `README.md:228`:

```
- [Atlarix](https://atlarix.dev) - Native desktop AI coding environment
  with Blueprint RAG and multi-provider support including Ollama
```

Slot is correctly placed right after Open Interpreter and before
the "Libraries & SDKs" section header — matches the existing
positional convention (alphabetical-ish, but really
chronological-by-PR).

## Risks

- **Link verifiability.** `atlarix.dev` is a third-party domain.
  Project README links don't carry SLA expectations but a dead
  link would be a maintenance papercut down the road. Worth the
  reviewer doing a quick `curl -I https://atlarix.dev` before
  merging.
- **Ollama integration claim.** PR doesn't link to a public repo,
  source file, or docs page demonstrating the Ollama integration.
  Most adjacent entries (QodeAssist, Open Interpreter) link to
  GitHub repos or docs paths, not just a marketing landing page.
  This is the only nit.
- **Marketing-language tone.** "Native desktop AI coding
  environment with Blueprint RAG and multi-provider support
  including Ollama" is reasonably restrained — comparable in tone
  to neighbors like "AI coding assistant for Qt Creator". OK as-is.

## Suggestions

- Swap the bare landing-page link for a deeper link if Atlarix
  has a docs/Ollama-setup page (mirrors what Open Interpreter
  does: `docs.openinterpreter.com/.../ollama`). If no such page
  exists yet, the landing page is acceptable.
- Optional: drop "Native" — the word doesn't disambiguate against
  any other entry on the list and reads as marketing. "Desktop AI
  coding environment with..." is enough.

## What I learned

Integration-list READMEs are the cheapest community-contribution
surface in OSS — a single-line PR that adds nominal review
overhead and meaningful discovery value to the project. The right
review bar is "is this a real, working integration" rather than
deep technical scrutiny; the gate is link-liveness + tone, not
architecture.
