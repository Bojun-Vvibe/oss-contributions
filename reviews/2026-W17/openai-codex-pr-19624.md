---
pr: 19624
repo: openai/codex
sha: a00d72279f333705a743406fad3f44ef181ec0c9
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex #19624 — fix(skills): make budget warning actionable

- **Author**: fcoury-oai
- **Head SHA**: a00d72279f333705a743406fad3f44ef181ec0c9
- **Size**: ~367 diff lines across `codex-rs/core-skills/src/render.rs`, `codex-rs/app-server/tests/suite/v2/turn_start.rs`, plus snapshot/test fixtures.

## Scope

Skills-metadata budget warnings used to report aggregate stats only ("7 additional skills were not included"). Users had no way to know *which* skills were lost or *which* descriptions were truncated. This PR rewrites the warning so:
- **Description-only truncation** ("All skills are still available, but some descriptions were shortened.") lists the largest-truncated description names.
- **Skill omission** ("Some skills were omitted from the model-visible skills list.") lists the omitted skill names with an `, and N more` suffix capped at `MAX_WARNING_SKILL_NAMES = 5`.

## Specific findings

- `codex-rs/core-skills/src/render.rs:20` — new const `MAX_WARNING_SKILL_NAMES: usize = 5`. Reasonable cap; deserves a doc-comment explaining "5 names + suffix keeps the warning under one terminal line at 80 cols".
- `codex-rs/core-skills/src/render.rs:121-128` — `SkillRenderReport` gains `largest_truncated_description_names: Vec<String>` and `omitted_skill_names: Vec<String>`. The names are populated upstream; this PR only changes the report shape and warning formatting.
- `codex-rs/core-skills/src/render.rs:210-228` — the `if report.omitted_count > 0` branch is rewritten to use the new `warning_skill_names_suffix(...)` helper. The old singular/plural and `was`/`were` agreement code is gone — good simplification, since the new format ("Some skills were omitted") sidesteps singular/plural entirely.
- `codex-rs/core-skills/src/render.rs:285-296` — `warning_skill_names_suffix(label, names, total_count)` joins names with `, ` then appends `, and {hidden_count} more` only when `hidden_count = total_count - names.len() > 0`. Logic is right. Edge case to test: `names.len() == total_count` should produce no "and N more" tail (verified by the `if hidden_count > 0` guard).
- `codex-rs/core-skills/src/render.rs:269-284` — `budget_warning_prefix` simplified from `replacen` string-mangling to a clean `match` returning `&'static str`. Net deletion of the two long `pub const ..._WARNING_PREFIX: &str` constants. Good — the old approach of building one prefix and rewriting it was confusing. Only concern: any external consumer (tests in *other* crates, grep-based log scrapers) that pattern-matched on `"Warning: Exceeded skills context budget"` will break. The integration test at `app-server/tests/suite/v2/turn_start.rs:334-337` was updated; verify no other public references.
- `codex-rs/app-server/tests/suite/v2/turn_start.rs:334-337` — the assertion now pins the new format including the actual omitted skill names (`imagegen, openai-docs, plugin-creator, skill-creator, skill-installer, and 2 more`). This is good as a behavioral pin but also brittle — if the underlying skill discovery order changes, this test breaks. Consider asserting on prefix + count rather than the literal concatenated names.

## Risk

Low. Pure UX text change with structural support for a richer report. No protocol/wire changes. Risk is downstream consumers grepping the old `"Warning: Exceeded skills context budget"` literal — release-notes-worthy.

## Verdict

**merge-after-nits** — call out the warning-text contract change in release notes (downstream log scrapers will break), consider loosening the integration assertion to prefix+count, and add a brief comment justifying `MAX_WARNING_SKILL_NAMES = 5`. The user-visible improvement is clearly worth shipping.
