# sst/opencode #24852 — fix: use JSON skill serialization by default for non-Anthropic models

- PR: https://github.com/sst/opencode/pull/24852
- Head SHA: `8cb3f42317b454fc24580337bb68f75de38f404c`
- Files: `packages/opencode/src/config/skills.ts`, `packages/opencode/src/session/prompt.ts`, `packages/opencode/src/session/system.ts`, `packages/opencode/src/skill/index.ts`, `packages/opencode/src/tool/registry.ts`, `packages/opencode/test/session/system.test.ts`

## Citations

- `packages/opencode/src/skill/index.ts` `fmt()` rewritten — old `opts: { verbose: boolean }` becomes `opts: { format: "xml" | "json" | "markdown" }`, with explicit branches for each (JSON branch builds a structured `{available_skills: [{name, description, location}]}` object).
- `packages/opencode/src/session/system.ts` line ~36 adds `isAnthropicModel(model)` (`providerID === "anthropic" || api.id.includes("claude")`) and the `skills()` method now takes `model: Provider.Model` and resolves `format = cfg.skills?.format ?? (isAnthropic ? "xml" : "json")`.
- `packages/opencode/src/config/skills.ts` adds `format: Schema.Union(Literal("xml"|"json"|"markdown"))` as opt-in user override.
- `packages/opencode/src/tool/registry.ts:260` switches the *tool-description* surface from `verbose: false` to the explicit `format: "markdown"` — this is the third format and only used here, deliberately distinct from the system-prompt surface.
- `test/session/system.test.ts` covers both Anthropic XML (legacy test renamed) and a new Ollama JSON test that parses the output and asserts ordering.

## Verdict

`merge-after-nits`

## Reasoning

The framing of the bug is correct: `<available_skills><skill>...</skill></available_skills>` reads as a strong format signal to non-Claude models, and devstral / qwen / llama variants will mirror it back as XML in their replies. The fix space here had three reasonable shapes — drop the prompt entirely, switch to natural-language prose, or switch to a structured-but-neutral format — and JSON is the right pick for general models because it's the same training-data shape they see in tool-call surfaces, so it doesn't get echoed in prose answers.

What's well-shaped: the discriminator `format` parameter replaces a boolean (`verbose`) which had become a load-bearing two-bit decision; "tool-description surface = markdown" / "system-prompt surface = xml or json" is now legible at every call site instead of inferred from `verbose: true|false`. The Anthropic detection conjunction (`providerID === "anthropic" || api.id.includes("claude")`) is right — Bedrock/Vertex Claude routes through other providerIDs but always carries `claude` in the API id, so neither prong alone is sufficient. The new test loads JSON from `result!.split("\n").slice(2).join("\n")` because the helper prepends two natural-language lines; that's brittle but acceptable since a future change to the preamble would surface as a localized test fix rather than silent breakage.

Nits worth raising before merge: (1) the `format: "markdown"` surface in `tool/registry.ts:260` is *only* used by the tool description — if a user sets `cfg.skills?.format = "markdown"` it currently gets ignored at the system-prompt site (the new `system.ts` only branches between Anthropic-default-xml and others-default-json, never reaches the markdown branch unless the user sets it). That's probably intentional but the config schema's description should say so. (2) The JSON branch emits `pathToFileURL(skill.location).href` — Anthropic XML did the same, but for JSON output sent to a *local Ollama* model, exposing absolute filesystem URIs in every prompt is a small fingerprinting/privacy regression worth noting; consider truncating to basename for the JSON branch since local models won't `Read` the SKILL.md anyway. (3) No test asserts `format: "markdown"` round-trips as the *system prompt* output (only the tool-description path), so the markdown branch in `fmt()` is effectively unreached at runtime — either delete it or add a config-override test.

The change is correct and well-scoped; the nits are post-merge polish.
