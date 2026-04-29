# sst/opencode#24976 — docs(providers): add Perplexity (Agent API + Search API)

- **Repo:** sst/opencode
- **PR:** #24976
- **Head SHA:** 20cec1777fe1614168903e32b36ca77ac4a42afe
- **Base:** dev
- **Author:** community

## Summary

Pure-docs change adding a new "Perplexity" section to `packages/web/src/content/docs/providers.mdx` between OpenCode Zen and LLM Gateway. Documents the OpenAI-Responses-compatible Agent API path (configured via `@ai-sdk/openai` with `baseURL: "https://api.perplexity.ai/v1"`) with three example models (`sonar-pro`, `sonar-reasoning-pro`, `sonar`), plus a Search API section showing both direct `curl` usage and an MCP-server config pointing at `https://mcp.perplexity.ai/sse` with bearer-token auth.

## File:line references

- `packages/web/src/content/docs/providers.mdx:1645-1707` — new Perplexity Agent API section (provider config snippet)
- `packages/web/src/content/docs/providers.mdx:1708-1741` — Search API subsection (curl example + MCP config)
- 97 additions, 0 deletions, single file touched

## Verdict: **merge-after-nits**

## Rationale

Docs-only, low risk, follows the established provider-section template (OpenCode Zen above, LLM Gateway below). Three nits:

1. **Model-list freshness risk.** Hardcoding `openai/gpt-5.4` and `anthropic/claude-sonnet-4-6` as routed-third-party examples at `providers.mdx:1683` will age badly — Perplexity rotates routed models frequently. Either drop the specific model names and link to the Perplexity models page, or add an "as of <date>" hedge so a future reader knows to verify.
2. **Token limits should be sourced.** The `"limit": { "context": 200000, "output": 8192 }` values for `sonar-pro` at `providers.mdx:1666` are not cited. If Perplexity tightens output to 4K next quarter, opencode users with this exact config will get truncation surprises. Add a comment line like `// Verify against https://docs.perplexity.ai/docs/agent-api/models — these were current as of 2026-04` or link to the upstream limits doc.
3. **MCP example should mention scope.** The MCP config at `providers.mdx:1730-1740` enables search as a tool but doesn't note that the agent can now consume Perplexity tokens on every turn (cost surprise) — add one line warning users to set per-tool budgets if they're cost-sensitive, and clarify whether `headers.Authorization` is interpolated by opencode at config-load time or per-request (matters for credential rotation).
4. Optional: the curl example at `providers.mdx:1716-1725` uses `--data` with a bash heredoc-style JSON literal — fine for docs, but consider showing the equivalent invocation through opencode's MCP path so users don't think direct HTTP is the only way.
