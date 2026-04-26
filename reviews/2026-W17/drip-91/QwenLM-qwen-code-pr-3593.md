---
pr: 3593
repo: QwenLM/qwen-code
sha: 342bad9ee0435ca82abba14fff2213f06575e0d7
verdict: merge-as-is
date: 2026-04-27
---

# QwenLM/qwen-code #3593 — feat(cli): Add argument-hint support for slash commands

- **Head SHA**: `342bad9ee0435ca82abba14fff2213f06575e0d7`
- **URL**: https://github.com/QwenLM/qwen-code/pull/3593
- **Size**: medium (259/20 across 24 files; most are tests)

## Summary
Threads an `argumentHint` field through every slash-command source: ACP integration, bundled skills, file-based markdown commands, skill-derived commands, and the autocomplete UI. When set, the hint shows up in the ACP `availableCommands` payload as `input: { hint: "..." }` and presumably surfaces in the TUI prompt next to the command name.

## Specific findings
- `packages/cli/src/acp-integration/session/Session.ts:984` — the change is a clean conditional: `input: cmd.argumentHint ? { hint: cmd.argumentHint } : null`. Preserves the old `null` shape when no hint is set, so existing ACP clients that pattern-match on `input === null` keep working. This is the right backward-compat choice.
- `packages/cli/src/acp-integration/session/Session.test.ts:223-243` — updated to assert that a command with `argumentHint: '[path]'` round-trips through to `input: { hint: '[path]' }`. Test is small and exact.
- `packages/cli/src/services/BundledSkillLoader.ts:69` — `argumentHint: skill.argumentHint` propagation. Pure passthrough; no transformation. Matches the pattern used for `whenToUse` in the same struct.
- `packages/cli/src/services/FileCommandLoader.ts:358` — `argumentHint: validDef.frontmatter?.['argument-hint']`. Bracket-key lookup is correct (kebab-case in YAML frontmatter, camelCase in TS). The TOML/YAML frontmatter parsers happily produce kebab-case keys; this is consistent with how `disable-model-invocation` is handled two lines below.
- `packages/cli/src/services/FileCommandLoader-markdown.test.ts:30,51` — markdown frontmatter test gets a new `argument-hint: "[issue-number]"` line and a `.toBe('[issue-number]')` assertion. Good.
- `packages/cli/src/services/BundledSkillLoader.test.ts:72-80` — new dedicated test for "should propagate argumentHint from bundled skills to slash commands". Uses `makeSkill({ argumentHint: '[topic]' })` and `expect(commands[0]?.argumentHint).toBe('[topic]')`. Tight scope.
- The 24-file scope is larger than the conceptual change but justifiable: any place that constructs a `SlashCommand` had to learn the new optional field, and the test fixtures had to be updated. The vast majority of the diff is mechanical "thread a new optional field through" changes, all of which preserve the prior `undefined` semantics by being optional.
- One thing not visible in the diff sample but worth verifying on the full PR: whether the autocomplete UI actually renders the hint, or whether it's only consumed by ACP. If only ACP, the PR title's "for slash commands" is slightly aspirational.

## Risk
Low. Every new field is optional with `undefined` as the default-equivalent, so any downstream consumer that hasn't been updated keeps its previous behavior. ACP wire format keeps `input: null` when the hint isn't set.

## Verdict
**merge-as-is** — clean, mechanical threading of an optional UX field with appropriate test coverage at every layer. Backward compatible with ACP clients that pattern-match on `input === null`.
