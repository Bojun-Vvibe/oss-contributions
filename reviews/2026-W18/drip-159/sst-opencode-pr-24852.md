# Review: sst/opencode#24852 — fix: use JSON skill serialization by default for non-Anthropic models

- **Author**: andrewgwoodruff
- **Head SHA**: `8cb3f42317b454fc24580337bb68f75de38f404c`
- **Diff**: +106 / −25 across 6 files
- **Closes**: #24824
- **Draft**: no

## What it does

OpenCode's system-prompt builder serializes the available-skills catalog into the prompt. Pre-PR, `Skill.fmt(list, { verbose: true })` always emitted XML (`<available_skills>/<skill>/<name>/<description>/<location>`). Anthropic Claude is trained on dense XML and tolerates this; non-Anthropic models (notably local Ollama models like devstral:24b) interpret an XML-shaped system prompt as "respond in XML" and break their JSON tool-call output. The PR author measured a real delta on devstral:24b: 50 trials per format, 68% pass on XML vs 86% on JSON (74% on Markdown).

The fix is a four-touch surgical change:

1. `config/skills.ts:12-15` adds a new optional `format: "xml" | "json" | "markdown"` config knob with a description that hard-codes the new default rule.
2. `session/system.ts:36-90` adds an `isAnthropicModel(model)` predicate that matches `providerID === "anthropic"` *or* `model.api.id.includes("claude")`, threads the `model: Provider.Model` argument through `SystemPrompt.Service.skills()`, computes `format = cfg.skills?.format ?? (isAnthropicModel(model) ? "xml" : "json")`, and calls `Skill.fmt(list, { format })`. `defaultLayer` now also `Layer.provide`s `Config.defaultLayer`.
3. `skill/index.ts:264-298` rewrites `fmt(list, opts)` from `{ verbose: boolean }` to `{ format: "xml" | "json" | "markdown" }`, with a JSON branch that emits `{ available_skills: [{ name, description, location: pathToFileURL(...).href }, ...] }` via `JSON.stringify(..., null, 2)`, the existing XML branch, and the existing markdown branch (now reachable via `format: "markdown"`). All three branches sort by `name.localeCompare`.
4. `tool/registry.ts:260` updates the in-tool-description rendering to `Skill.fmt(list, { format: "markdown" })` (was `verbose: false`, which already produced markdown — semantic-preserving rename).
5. `session/prompt.ts:1443` passes `model` through to `sys.skills(agent, model)`.
6. `test/session/system.test.ts` adds 64 lines: keeps the existing stability test (now passing the new `anthropicModel` fixture and asserting XML output), plus a new `"skills output uses JSON format for non-Anthropic models"` test that creates an `ollamaModel` fixture (`providerID: "ollama"`, `api.id: "devstral:24b"`), invokes `svc.skills(build!, ollamaModel)`, slices off the two preamble lines, `JSON.parse`s the rest, and asserts both `available_skills[].name` order and JSON-shape.

## Concerns

1. **`isAnthropicModel` predicate is a substring sniff that will misfire on third-party `claude*` names.** `model.api.id.includes("claude")` matches `"openrouter/anthropic/claude-3.5-sonnet"` (correct) but also hypothetical re-hosters that put `"claude"` in the model id without serving an Anthropic API (e.g. a Bedrock model id `"anthropic.claude-3-5-sonnet-20241022-v2:0"` — actually Anthropic-shaped, so correct here, but a self-hosted finetune named `"claude-distilled-7b"` on Ollama would route to XML and lose the very gain this PR is shipping). The `providerID === "anthropic"` half of the OR is the principled half; the substring fallback is the half that will need maintenance. Worth narrowing to a hardcoded provider allowlist (`anthropic`, `bedrock`, `vertex_ai-anthropic`, `openrouter` when paired with an `anthropic/` namespace) or at least hoisting a `// substring fallback handles bedrock/vertex/openrouter routing of Anthropic models — may misclassify look-alike model ids, override via skills.format` comment.

2. **No fallback for the third format gap.** Anyone running a *Markdown-preferring* model (some open-source models do markedly better on markdown than JSON) has to set `skills.format` explicitly. The author's own data shows markdown at 74% on devstral, splitting the diff between XML 68% and JSON 86% — JSON wins for devstral, but the PR description doesn't generalize. A short matrix recommendation in the config description (or in docs) would help users not have to re-discover the format-vs-model interaction.

3. **JSON test parses `result!.split("\n").slice(2).join("\n")`.** This couples the test to the exact "two preamble lines, then the JSON" wire format. If anyone ever changes the preamble to one line or three (or wraps with a markdown fence), the test breaks for cosmetic reasons rather than substantive ones. A regex or a "find the first `{`" search would be more robust.

4. **`defaultLayer` now depends on `Config.defaultLayer`.** Anything that previously composed `SystemPrompt.defaultLayer` standalone now drags a Config dependency. The grep for external consumers isn't shown in the diff — worth a quick check that test fixtures and any embed scenarios don't break.

5. **`Skill.fmt` API breaking rename.** The `{ verbose: boolean }` → `{ format: "xml" | "json" | "markdown" }` change is unguarded — any external caller using `verbose: true/false` will silently fall through to the markdown default at the bottom of the function. This is internal API today, but if the package is consumed externally this is a semver breaker that needs a CHANGELOG note.

## Verdict

**merge-after-nits** — measured-delta-driven change with a real test surface and a clean override path via `skills.format`. Tighten the Anthropic-detection heuristic (or comment it explicitly), fix the brittle preamble-counting in the test, and confirm `Skill.fmt`'s shape change has no external consumers.

