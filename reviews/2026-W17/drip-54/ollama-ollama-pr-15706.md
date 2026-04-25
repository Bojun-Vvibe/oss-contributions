# ollama/ollama PR #15706 — Add LumaKit to Community Integrations

@9d5fe725 · base `main` · +1/-0 · author `patmakesapps`

## Summary
One-line README addition under "Frameworks & Agents" linking to LumaKit, a local-first agent.

## What changed
- `README.md:271` — single line: `- [LumaKit](https://github.com/patmakesapps/LumaKit) - Local-first AI agent for Ollama with a web UI, launcher flow, persistent tools, browser automation, reminders, and autonomous tasks`

## Key observations
- Inserted in the correct alphabetical-ish neighborhood (after Neuro SAN, before the RAG subsection at `README.md:272`). List is not strictly alphabetized so position is fine.
- Description is on the long side compared to neighbors (Stakpak: 5 words; Hexabot: 4 words). Trimming to "Local-first AI agent with launcher, browser automation, reminders, and persistent tools" would match the section's voice.
- Linked repo `github.com/patmakesapps/LumaKit` should be sanity-checked by a maintainer for content/license; this review didn't fetch it.
- No CI/code impact; safe trivial change.

## Risks/nits
- Style nit only; no functional risk.

**Verdict: merge-after-nits**
