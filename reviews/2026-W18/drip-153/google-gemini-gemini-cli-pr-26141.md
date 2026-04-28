# google-gemini/gemini-cli #26141 — fix(core): add missing oauth fields support in subagent parsing

- PR: https://github.com/google-gemini/gemini-cli/pull/26141
- Head SHA: `f0c377bfc1c345736bbe05fdd618816d3ef2ee9a`
- Files: `packages/core/src/agents/agentLoader.ts`, `packages/core/src/agents/agentLoader.test.ts`, `packages/core/src/agents/auth-provider/types.ts`

## Citations

- `agentLoader.ts:79-89` — inline `mcp_servers` `auth` schema for the `oauth` discriminant adds five fields: `issuer`, `audiences` (`z.array(z.string())`), `redirect_uri`, `token_param_name`, `registration_url`.
- `agentLoader.ts:153-163` — top-level `oauth2AuthSchema` mirrors the same five additions, keeping the two zod schemas in lockstep.
- `agentLoader.ts:469-478` — `convertFrontmatterAuthToConfig` now propagates the new fields into the `oauth2`-shape config (snake_case preserved at this boundary).
- `agentLoader.ts:567-577` — `markdownToAgentDefinition` also propagates them but renames to camelCase (`redirectUri`, `tokenParamName`, `registrationUrl`) for the runtime `MCPOAuthConfig` shape.
- `auth-provider/types.ts:80-83` — `OAuth2AuthConfig` interface gets the five new optional fields.
- `agentLoader.test.ts:530+` (`should convert mcp_servers with auth block in local agent (oauth with full fields)`) — full positive round-trip with all 11 oauth fields populated, asserting the snake→camel rename happens correctly.
- `agentLoader.test.ts:888+` — the existing `parseAgentMarkdown` YAML test is extended to cover the new fields end-to-end through the markdown parser.

## Verdict

`merge-after-nits`

## Reasoning

This is a small, mechanical, fill-the-gap PR that brings the inline subagent schema in line with the canonical `MCPOAuthConfig` schema used elsewhere in the repo. The bug it fixes is real — a user writes an `agent.md` with the full OAuth block (because the docs for `MCPOAuthConfig` show those fields) and the inline parser silently drops `issuer`, `audiences`, `redirect_uri`, etc. on the floor, leading to mysterious "OAuth flow doesn't work" reports. The fix is the obvious one: add the fields in three places (zod schema for validation, frontmatter→config converter, markdown→runtime converter) and add tests.

Implementation quality is fine. The double-schema pattern (inline `mcp_servers.auth` discriminator inside `mcpServerSchema` *and* the standalone `oauth2AuthSchema`) is unfortunate but pre-existing — the PR doesn't compound it, just adds the same five fields to both. Worth a separate refactor PR to define `oauth2AuthSchema` once and reference it from `mcpServerSchema`, but that's not this PR's job.

Two nits worth raising before merge:

1. **Test asymmetry around the dual schema.** The new test at `agentLoader.test.ts:530+` covers the local-agent inline-mcp-servers path (`markdownToAgentDefinition`). The new YAML lines at `agentLoader.test.ts:888+` cover the standalone `oauth` auth-type path through `parseAgentMarkdown`. Both are exercised, which is good. But there's no test that asserts the *two parser paths produce the same shape* for the same input — i.e. that a given OAuth block in either location ends up with identical field names and types in the final config. Given that this PR is fixing a drift between two parsers, a parity test would prevent the next drift.

2. **No negative test for unknown fields.** zod's default behavior strips unknown keys silently. A user who mistypes `redirect_uri` as `redirectUri` in their YAML frontmatter will get the same silent-drop bug this PR is fixing, just one layer over. Adding `.strict()` to the auth shape (or a runtime warning when unknown keys are present) would harden the parser against the same class of issue. Out of scope for this PR but worth a follow-up.

3. **`audiences: z.array(z.string()).optional()`** — RFC 7519 / OAuth 2.0 also accepts `aud` as a single string, not just an array. Some OPs emit `"audiences": "https://api.example.com"` rather than `["https://api.example.com"]`. Consider `z.union([z.string(), z.array(z.string())]).optional()` and normalize to array post-parse. Not blocking — just observed.

The change closes #24836 cleanly. Ship after considering the parity test.
