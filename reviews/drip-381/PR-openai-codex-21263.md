# openai/codex PR #21263 — [codex] Coordinate OpenAI docs sample with API key setup

- URL: https://github.com/openai/codex/pull/21263
- Head SHA: `826fecf8a7b7df6a0b3fafa24aabcac2bda3755b`
- Size: +6 / -0 (single SKILL.md doc edit)

## Summary

Six-line documentation insertion into `codex-rs/skills/src/assets/samples/openai-docs/SKILL.md:11-16` that adds an "API Key Setup" section telling the agent to delegate to `openai-platform-api-key` first when the user is building/running/configuring an API-backed app, then return to this skill for docs. Clarifies the docs-only fallback path (citations, conceptual explanations, examples that don't require building or running an API-backed artifact).

## Specific findings

- `SKILL.md:11-12` — section header `## API Key Setup` placed immediately after the existing one-paragraph skill description and before the existing `## Quick start` heading. Position is correct: it's a *gating* instruction for skill dispatch, so it belongs above the operational `Quick start`.
- `SKILL.md:13` — directive: "For requests to build, run, configure, debug, or implement an API-backed app, script, CLI, generator, or tool, use `openai-platform-api-key` first when available." The verb list (build/run/configure/debug/implement) and noun list (app/script/CLI/generator/tool) is broad and overlapping — basically a denylist for "anything that needs a real API key at runtime." Reasonable scope.
- `SKILL.md:13` — qualifier "when available" hedges correctly: if the `openai-platform-api-key` skill isn't installed, this instruction degrades to a no-op rather than blocking the doc-fetch path.
- `SKILL.md:15` — fallback clause: "Use this skill directly for docs-only questions, citations, model/API guidance, conceptual explanations, and examples that do not require building or running an API-backed artifact." Captures the inverse of the gating list cleanly.

## Notes

- Pure prose change to a sample skill template; no code, no tests required.
- "After that credential gate is resolved, return here for current docs as needed" is a reasonable hand-off pattern — assumes the agent runtime supports skill chaining (which the codex skill loader does).
- One nit: the gating bullet doesn't say what to do if `openai-platform-api-key` is *available but the user already has a key configured*. Implicit answer is "the other skill no-ops and you proceed here," but the SKILL.md author may want to make that explicit to head off agent loops where both skills bounce control back and forth.

## Verdict

`merge-as-is`
