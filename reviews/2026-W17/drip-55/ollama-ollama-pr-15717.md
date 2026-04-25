# ollama/ollama PR #15717 — Add Spring AI Playground to Chat Interfaces (Desktop)

@11757ef7 · base `main` · +1/-0 · author `JM-Lab`

## Summary
Adds one bullet to the Community Integrations list pointing at Spring AI Playground, a desktop MCP client.

## What changed
- `README.md:204` — single line added under `Community Integrations > Chat Interfaces > Desktop`:
  ```
  - [Spring AI Playground](https://github.com/spring-ai-community/spring-ai-playground) - Desktop app with MCP tool builder, RAG, and agentic chat
  ```

## Key observations
- Pure README addition; no code, no risk. The link target is a real spring-ai-community repo and the description is short and factual.
- Alphabetical ordering in this section is loose (the surrounding entries — Kerlig, Hillnote, Perfect Memory — are not sorted), so position is fine. If maintainers prefer sorted, ask author to re-place; otherwise leave.
- Description hits the three concrete features (MCP tool builder, RAG, agentic chat) rather than marketing fluff, which matches the tone of nearby entries.

## Risks/nits
- Self-promotion check: author `JM-Lab` is a member of `spring-ai-community`. That's standard for this list; the project itself looks active and on-topic.
- No need for a maintainer to verify install steps — the link is the contract.

**Verdict: merge-as-is**
