# Review Index

24 PR reviews across 8 OSS AI-coding-agent projects, 3 per repo.
Each review contains: context, diff walkthrough with quoted snippets,
risks, suggested questions, verdict, and a "what I learned" section.

## Aider-AI/aider

| PR | Title | File |
|---|---|---|
| [#4855](https://github.com/Aider-AI/aider/pull/4855) | feat: Add native EUrouter provider support | [PR-4855.md](Aider-AI-aider/PR-4855.md) |
| [#4858](https://github.com/Aider-AI/aider/pull/4858) | feat: add double-buffer context management | [PR-4858.md](Aider-AI-aider/PR-4858.md) |
| [#4936](https://github.com/Aider-AI/aider/pull/4936) | Add ACP (Agent Communication Protocol) support adapter | [PR-4936.md](Aider-AI-aider/PR-4936.md) |

## All-Hands-AI/OpenHands

| PR | Title | File |
|---|---|---|
| [#13977](https://github.com/All-Hands-AI/OpenHands/pull/13977) | Warn before resetting hidden SDK settings | [PR-13977.md](All-Hands-AI-OpenHands/PR-13977.md) |
| [#13983](https://github.com/All-Hands-AI/OpenHands/pull/13983) | feat(app-server): route ACP agents to the ACP conversation endpoint | [PR-13983.md](All-Hands-AI-OpenHands/PR-13983.md) |
| [#13994](https://github.com/All-Hands-AI/OpenHands/pull/13994) | feat(frontend): render ACPToolCallEvent in conversation viewer | [PR-13994.md](All-Hands-AI-OpenHands/PR-13994.md) |

## anomalyco/opencode

| PR | Title | File |
|---|---|---|
| [#23755](https://github.com/anomalyco/opencode/pull/23755) | fix: preserve thinking/redacted_thinking blocks in Anthropic message transforms | [PR-23755.md](anomalyco-opencode/PR-23755.md) |
| [#23862](https://github.com/anomalyco/opencode/pull/23862) | fix: sessions missing from sidebar on Windows due to path separator mismatch | [PR-23862.md](anomalyco-opencode/PR-23862.md) |
| [#23866](https://github.com/anomalyco/opencode/pull/23866) | fix(project): use worktree paths for project ids | [PR-23866.md](anomalyco-opencode/PR-23866.md) |

## BerriAI/litellm

| PR | Title | File |
|---|---|---|
| [#26274](https://github.com/BerriAI/litellm/pull/26274) | fix(mcp): harden OAuth authorize/token endpoints (BYOK + discoverable) | [PR-26274.md](BerriAI-litellm/PR-26274.md) |
| [#26279](https://github.com/BerriAI/litellm/pull/26279) | fix(auth): centralize common_checks to close authorization bypass | [PR-26279.md](BerriAI-litellm/PR-26279.md) |
| [#26285](https://github.com/BerriAI/litellm/pull/26285) | fix(anthropic): preserve reasoning content and add think-tag regression coverage | [PR-26285.md](BerriAI-litellm/PR-26285.md) |

## charmbracelet/crush

| PR | Title | File |
|---|---|---|
| [#2579](https://github.com/charmbracelet/crush/pull/2579) | feat(tool): add `ask-user-questions` tool | [PR-2579.md](charmbracelet-crush/PR-2579.md) |
| [#2613](https://github.com/charmbracelet/crush/pull/2613) | fix(agent): prune excess images from history to prevent session deadlock | [PR-2613.md](charmbracelet-crush/PR-2613.md) |
| [#2663](https://github.com/charmbracelet/crush/pull/2663) | fix(app): replace single events channel with pubsub.Broker for fan-out | [PR-2663.md](charmbracelet-crush/PR-2663.md) |

## cline/cline

| PR | Title | File |
|---|---|---|
| [#10210](https://github.com/cline/cline/pull/10210) | Remove `/deep-planning` built-in slash command | [PR-10210.md](cline-cline/PR-10210.md) |
| [#10254](https://github.com/cline/cline/pull/10254) | fix: use deterministic keys for MCP server tool routing | [PR-10254.md](cline-cline/PR-10254.md) |
| [#10329](https://github.com/cline/cline/pull/10329) | feat(cost-control): enforce third-party API spend limits [PF-672] | [PR-10329.md](cline-cline/PR-10329.md) |

## modelcontextprotocol/servers

| PR | Title | File |
|---|---|---|
| [#3890](https://github.com/modelcontextprotocol/servers/pull/3890) | feat: Add compare_directories tool for directory comparison | [PR-3890.md](modelcontextprotocol-servers/PR-3890.md) |
| [#3922](https://github.com/modelcontextprotocol/servers/pull/3922) | fix(fetch): fall back when Readability strips hidden SSR content | [PR-3922.md](modelcontextprotocol-servers/PR-3922.md) |
| [#3959](https://github.com/modelcontextprotocol/servers/pull/3959) | feat(memory): add lightweight read modes for scalability | [PR-3959.md](modelcontextprotocol-servers/PR-3959.md) |

## openai/codex

| PR | Title | File |
|---|---|---|
| [#18868](https://github.com/openai/codex/pull/18868) | Add MITM hooks for host specific HTTPS request clamping | [PR-18868.md](openai-codex/PR-18868.md) |
| [#19030](https://github.com/openai/codex/pull/19030) | Support to add prefetched tool result to a user turn | [PR-19030.md](openai-codex/PR-19030.md) |
| [#19058](https://github.com/openai/codex/pull/19058) | Add /auto-review-denials retry approval flow | [PR-19058.md](openai-codex/PR-19058.md) |

---

See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes that emerged
across the 24 reviews.
