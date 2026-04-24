# openai/codex PR #19407 — Update bundled OpenAI Docs skill for GPT-5.5

- **Author:** kkahadze-oai
- **Head SHA:** eeaa70310aca24c36e0e67fab6177d74bba3ee7f
- **Files:** 6 under `codex-rs/skills/src/assets/samples/openai-docs/`
  (+214 / −554)
- **Verdict:** `merge-after-nits`

## Context

The `openai-docs` skill is bundled into the codex CLI as a sample.
It carries its own `SKILL.md`, agent manifest, fallback reference docs
(`latest-model.md`, `prompting-guide.md`, `upgrade-guide.md`), and a
helper script. With GPT-5.5 shipping, the skill needs new defaults,
new model recommendations, and a refreshed prompting guide. This PR
mostly rewrites `prompting-guide.md` and updates the model table.

## What the diff does

- `SKILL.md` line 9: removes the "may also load targeted files from
  references/" hedge and asserts the skill *owns* model selection,
  migration, and prompt-upgrade guidance. Quick-start bullet for
  the resolver script is narrowed to "only when target is
  latest/current/default."
- `agents/openai.yaml` lines 3 and 6: short description and
  default prompt updated to mention model selection and migration
  rather than upgrade-only framing.
- `references/latest-model.md` lines 7–18: model table replaced.
  `gpt-5.5` becomes the default; `gpt-5.4` is repositioned as
  "previous default … existing integrations." Adds `gpt-5.3-codex`
  for agentic coding and keeps `gpt-image-1.5`.
- `references/prompting-guide.md`: full rewrite — 599 → 244 lines —
  retargeted at GPT-5.5's "less-is-more, outcome-oriented" prompting
  style. New sections on personality, preamble for time-to-first-token,
  and collaboration style.

## Review notes

- The `latest-model.md` table at lines 9–18 lists `gpt-5.5` *twice*:
  once as default and again as "Explicit no-reasoning text path via
  `reasoning.effort: none`." That's confusing in a Markdown table
  the agent will quote verbatim — a single row with the
  `reasoning.effort` note in the "Use for" column would be clearer
  and avoid a duplicate-id appearance.
- `SKILL.md` line 17 narrowing of the resolver script is good — the
  prior text always invoked it and would clobber explicit user
  targets. Consider re-emphasising the "preserve explicit target"
  rule that already exists at line 18, since the script-invocation
  rule and the preserve-target rule are now in the same paragraph.
- The `prompting-guide.md` rewrite is a net improvement (much shorter,
  fewer brittle code-style mandates), but it drops the
  `<task_update>` / `<instruction_priority>` patterns that were load-
  bearing for mid-conversation steering. Worth retaining a one-
  paragraph pointer to those, since they're the documented escape
  hatch when a long-running agent needs to override earlier
  developer instructions.
- −554/+214 net is a healthy direction. The risk in shipping a
  shorter prompting guide is that fall-back-to-bundled-references
  degrades (when MCP is unreachable). The skill still mandates MCP
  primary, so this trade-off is reasonable.

## What I learned

When a skill ships its own fallback docs, every sentence in those
docs is effectively part of the model's context window every time
the skill activates. Aggressive trimming (this PR cuts the prompting
guide by ~60%) is usually the right call — fewer tokens, less
opportunity for stale advice to mislead. The duplicate row in the
model table is the kind of minor inconsistency that gets amplified
because the model will paraphrase it directly.
