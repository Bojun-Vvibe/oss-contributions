# QwenLM/qwen-code #3673 — feat(memory): add autoSkill background project skill extraction

- **Head SHA reviewed:** `e430ff9c2994a7b63c6d94baa68fc394c0087ccd`
- **Size:** +1840 / -23 across 10+ files
- **Verdict:** needs-discussion

## Summary

Adds an "AutoSkill" feature: when a session accumulates ≥ 20 tool
calls, a background review-agent fork distills reusable procedures
from the conversation into project-level skills under
`${projectRoot}/.qwen/skills/`. Off by default
(`memory.enableAutoSkill: false`).

The change is large (+1840) and touches agent runtime, settings
schema, non-interactive CLI, and adds a 433-line design doc under
`docs/design/skill-nudge/`.

## What I checked

- `docs/design/skill-nudge/skill-nudge.md:1-433` — well-written
  design doc. The decision to *not* introduce a `skill_manage`
  dedicated tool and instead reuse `write_file` / `edit` is sound,
  and the dual-protection (`source: auto-skill` frontmatter +
  `SkillScopedPermissionManager`) is the right shape. But the doc
  is in Chinese only; this repo's other design docs under
  `docs/design/` are bilingual or English-only. Add an English
  abstract at the top so non-Chinese reviewers can validate intent.
- The `auto-skill` frontmatter check is described as "权限管理器层
  (硬约束)" — implementation-level, but the diff doesn't show the
  actual permission-manager change in the snippet I sampled. Need
  to confirm `createSkillScopedAgentConfig` reads the frontmatter
  *before* allowing `edit`, not just at config-build time. A
  symlink-following or race condition between the frontmatter read
  and the `edit` write could allow override of a user-created skill.
- `packages/cli/src/config/settingsSchema.ts` (claimed in body) —
  ensure the new `memory.enableAutoSkill` key has a JSON-schema
  description so users see it in `/settings` discoverability.
- The 20-call threshold is hardcoded as `AUTO_SKILL_THRESHOLD`. The
  doc justifies "20 vs Hermes' 10" with reasoning about
  finer-grained tool calls in qwen-code, which is plausible. Make
  it user-overridable via settings (e.g.,
  `memory.autoSkillThreshold`), since users with custom tools may
  want very different defaults.
- `skillsModifiedInSession` is set to `true` whenever a write lands
  in `.qwen/skills/`. Consider whether *reads* of skills should
  also short-circuit (probably not — reads are common and skill
  review may want to update a skill it just read).
- The system prompt for the review agent (lines around the design
  doc's "触发 prompt" section) is a near-direct port of Hermes'
  `_SKILL_REVIEW_PROMPT`. Verify the upstream license / attribution
  is correctly noted (likely fine since both are MIT, but the
  source comment should explicitly cite Hermes).
- The non-interactive CLI tests are listed as touched
  (`packages/cli/src/nonInteractiveCli.test.ts`) — good, they
  should cover the case where `enableAutoSkill: true` triggers in
  a `qwen --no-interactive` flow without a TTY (background fork
  shouldn't try to render UI).

## Risk

Medium. The feature is opt-in, which limits blast radius. Real
risks are:

- Background fork may consume model quota silently after a long
  session ends. Document this prominently.
- The `source: auto-skill` protection is only as strong as the
  frontmatter parser; malformed YAML could cause either false
  positives (allowing edit of a user file) or false negatives.
- 1840 LOC in one PR is hard to bisect if regressions surface
  later.

## Recommendation

Needs-discussion before merge:

1. Confirm the frontmatter check is enforced at the **executor**
   level (right before write), not just at agent-config build
   time, to close the TOCTOU window.
2. Make the threshold a setting (`memory.autoSkillThreshold`).
3. Add an English abstract to the design doc.
4. Consider splitting: design doc as one PR, runtime wiring as a
   second, settings schema + CLI integration as a third — would
   make the feature reviewable and revertible piecewise.
5. Add an explicit "AutoSkill consumes additional model calls
   after each long session" line to the user-facing settings
   description and CHANGELOG.
