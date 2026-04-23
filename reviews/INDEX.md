# Review Index

48 PR reviews across 8 OSS AI-coding-agent projects, 6 per repo
(two batches of 3). Each review contains: context, problem, design
analysis with quoted snippets where useful, risks, suggestions,
verdict, and a "what I learned" section.

- **Round 1** (W7): the original 24, focused on feature additions
  and integration work.
- **W9 batch**: 24 more, focused on subtle correctness bugs —
  silent payload drops, deadlock recovery, cross-route loops,
  parity bugs between two surfaces sharing a backend.

See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.

## Aider-AI/aider

| PR | Title | File |
|---|---|---|
| [#4748](https://github.com/Aider-AI/aider/pull/4748) | Fix regression in the LiteLLM exception list | [PR-4748.md](Aider-AI-aider/PR-4748.md) |
| [#4830](https://github.com/Aider-AI/aider/pull/4830) | Allow adding files outside repo when git commits off | [PR-4830.md](Aider-AI-aider/PR-4830.md) |
| [#4838](https://github.com/Aider-AI/aider/pull/4838) | fix: handle symlink loops in safe_abs_path() | [PR-4838.md](Aider-AI-aider/PR-4838.md) |
| [#4855](https://github.com/Aider-AI/aider/pull/4855) | feat: Add native EUrouter provider support | [PR-4855.md](Aider-AI-aider/PR-4855.md) |
| [#4858](https://github.com/Aider-AI/aider/pull/4858) | feat: add double-buffer context management | [PR-4858.md](Aider-AI-aider/PR-4858.md) |
| [#4936](https://github.com/Aider-AI/aider/pull/4936) | Add ACP (Agent Communication Protocol) support adapter | [PR-4936.md](Aider-AI-aider/PR-4936.md) |

## All-Hands-AI/OpenHands

| PR | Title | File |
|---|---|---|
| [#13977](https://github.com/All-Hands-AI/OpenHands/pull/13977) | Warn before resetting hidden SDK settings | [PR-13977.md](All-Hands-AI-OpenHands/PR-13977.md) |
| [#13983](https://github.com/All-Hands-AI/OpenHands/pull/13983) | feat(app-server): route ACP agents to the ACP conversation endpoint | [PR-13983.md](All-Hands-AI-OpenHands/PR-13983.md) |
| [#13994](https://github.com/All-Hands-AI/OpenHands/pull/13994) | feat(frontend): render ACPToolCallEvent in conversation viewer | [PR-13994.md](All-Hands-AI-OpenHands/PR-13994.md) |
| [#14042](https://github.com/All-Hands-AI/OpenHands/pull/14042) | fix: prevent infinite redirect loop on org-defaults settings pages | [PR-14042.md](All-Hands-AI-OpenHands/PR-14042.md) |
| [#14044](https://github.com/All-Hands-AI/OpenHands/pull/14044) | fix(backend): repair org-defaults LLM save flow and sync managed keys to members | [PR-14044.md](All-Hands-AI-OpenHands/PR-14044.md) |
| [#14049](https://github.com/All-Hands-AI/OpenHands/pull/14049) | fix(frontend): restore notification sound and browser tab flash | [PR-14049.md](All-Hands-AI-OpenHands/PR-14049.md) |

## anomalyco/opencode

| PR | Title | File |
|---|---|---|
| [#23755](https://github.com/anomalyco/opencode/pull/23755) | fix: preserve thinking/redacted_thinking blocks in Anthropic message transforms | [PR-23755.md](anomalyco-opencode/PR-23755.md) |
| [#23797](https://github.com/anomalyco/opencode/pull/23797) | fix: preserve UTF-8 BOM through edit/patch operations | [PR-23797.md](anomalyco-opencode/PR-23797.md) |
| [#23837](https://github.com/anomalyco/opencode/pull/23837) | fix: fail fast on invalid session payloads | [PR-23837.md](anomalyco-opencode/PR-23837.md) |
| [#23862](https://github.com/anomalyco/opencode/pull/23862) | fix: sessions missing from sidebar on Windows due to path separator mismatch | [PR-23862.md](anomalyco-opencode/PR-23862.md) |
| [#23866](https://github.com/anomalyco/opencode/pull/23866) | fix(project): use worktree paths for project ids | [PR-23866.md](anomalyco-opencode/PR-23866.md) |
| [#23870](https://github.com/anomalyco/opencode/pull/23870) | fix: session compaction loss-of-context | [PR-23870.md](anomalyco-opencode/PR-23870.md) |

## BerriAI/litellm

| PR | Title | File |
|---|---|---|
| [#26219](https://github.com/BerriAI/litellm/pull/26219) | fix: ChatGPT responses bridge recovery | [PR-26219.md](BerriAI-litellm/PR-26219.md) |
| [#26228](https://github.com/BerriAI/litellm/pull/26228) | fix(anthropic): skip file-block discovery on unsupported content | [PR-26228.md](BerriAI-litellm/PR-26228.md) |
| [#26266](https://github.com/BerriAI/litellm/pull/26266) | fix(bedrock): guardrail logging + redaction | [PR-26266.md](BerriAI-litellm/PR-26266.md) |
| [#26274](https://github.com/BerriAI/litellm/pull/26274) | fix(mcp): harden OAuth authorize/token endpoints (BYOK + discoverable) | [PR-26274.md](BerriAI-litellm/PR-26274.md) |
| [#26279](https://github.com/BerriAI/litellm/pull/26279) | fix(auth): centralize common_checks to close authorization bypass | [PR-26279.md](BerriAI-litellm/PR-26279.md) |
| [#26285](https://github.com/BerriAI/litellm/pull/26285) | fix(anthropic): preserve reasoning content and add think-tag regression coverage | [PR-26285.md](BerriAI-litellm/PR-26285.md) |

## charmbracelet/crush

| PR | Title | File |
|---|---|---|
| [#2579](https://github.com/charmbracelet/crush/pull/2579) | feat(tool): add `ask-user-questions` tool | [PR-2579.md](charmbracelet-crush/PR-2579.md) |
| [#2613](https://github.com/charmbracelet/crush/pull/2613) | fix(agent): prune excess images from history to prevent session deadlock | [PR-2613.md](charmbracelet-crush/PR-2613.md) |
| [#2615](https://github.com/charmbracelet/crush/pull/2615) | fix(agent): validate tool call/results + strip tags from titles | [PR-2615.md](charmbracelet-crush/PR-2615.md) |
| [#2619](https://github.com/charmbracelet/crush/pull/2619) | fix(lsp): mitigate stale diagnostics | [PR-2619.md](charmbracelet-crush/PR-2619.md) |
| [#2622](https://github.com/charmbracelet/crush/pull/2622) | fix: inject synthetic tool_result for orphaned tool_use on session resume | [PR-2622.md](charmbracelet-crush/PR-2622.md) |
| [#2663](https://github.com/charmbracelet/crush/pull/2663) | fix(app): replace single events channel with pubsub.Broker for fan-out | [PR-2663.md](charmbracelet-crush/PR-2663.md) |

## cline/cline

| PR | Title | File |
|---|---|---|
| [#10210](https://github.com/cline/cline/pull/10210) | Remove `/deep-planning` built-in slash command | [PR-10210.md](cline-cline/PR-10210.md) |
| [#10254](https://github.com/cline/cline/pull/10254) | fix: use deterministic keys for MCP server tool routing | [PR-10254.md](cline-cline/PR-10254.md) |
| [#10266](https://github.com/cline/cline/pull/10266) | Fix cache reflection for Cline and Vercel handlers | [PR-10266.md](cline-cline/PR-10266.md) |
| [#10269](https://github.com/cline/cline/pull/10269) | fix: unblock stuck command_output ask when terminal command ends | [PR-10269.md](cline-cline/PR-10269.md) |
| [#10283](https://github.com/cline/cline/pull/10283) | feat: wire up remote globalSkills with enterprise UI and architectural fixes | [PR-10283.md](cline-cline/PR-10283.md) |
| [#10329](https://github.com/cline/cline/pull/10329) | feat(cost-control): enforce third-party API spend limits | [PR-10329.md](cline-cline/PR-10329.md) |

## modelcontextprotocol/servers

| PR | Title | File |
|---|---|---|
| [#3297](https://github.com/modelcontextprotocol/servers/pull/3297) | fix(memory): correct openNodes/searchNodes edge semantics | [PR-3297.md](modelcontextprotocol-servers/PR-3297.md) |
| [#3434](https://github.com/modelcontextprotocol/servers/pull/3434) | fix(filesystem): normalize bare drive letters on Windows | [PR-3434.md](modelcontextprotocol-servers/PR-3434.md) |
| [#3545](https://github.com/modelcontextprotocol/servers/pull/3545) | fix(git): guards against arg injection in git operations | [PR-3545.md](modelcontextprotocol-servers/PR-3545.md) |
| [#3890](https://github.com/modelcontextprotocol/servers/pull/3890) | feat: Add compare_directories tool for directory comparison | [PR-3890.md](modelcontextprotocol-servers/PR-3890.md) |
| [#3922](https://github.com/modelcontextprotocol/servers/pull/3922) | fix(fetch): fall back when Readability strips hidden SSR content | [PR-3922.md](modelcontextprotocol-servers/PR-3922.md) |
| [#3959](https://github.com/modelcontextprotocol/servers/pull/3959) | feat(memory): add lightweight read modes for scalability | [PR-3959.md](modelcontextprotocol-servers/PR-3959.md) |

## openai/codex

| PR | Title | File |
|---|---|---|
| [#18868](https://github.com/openai/codex/pull/18868) | Add MITM hooks for host specific HTTPS request clamping | [PR-18868.md](openai-codex/PR-18868.md) |
| [#19030](https://github.com/openai/codex/pull/19030) | Support to add prefetched tool result to a user turn | [PR-19030.md](openai-codex/PR-19030.md) |
| [#19031](https://github.com/openai/codex/pull/19031) | Fix relative stdio MCP cwd fallback | [PR-19031.md](openai-codex/PR-19031.md) |
| [#19046](https://github.com/openai/codex/pull/19046) | exec-server: require explicit filesystem sandbox cwd | [PR-19046.md](openai-codex/PR-19046.md) |
| [#19058](https://github.com/openai/codex/pull/19058) | Add /auto-review-denials retry approval flow | [PR-19058.md](openai-codex/PR-19058.md) |
| [#19086](https://github.com/openai/codex/pull/19086) | app-server: include filesystem entries in permission requests | [PR-19086.md](openai-codex/PR-19086.md) |

---

See [INSIGHTS.md](INSIGHTS.md) for cross-cutting themes.
