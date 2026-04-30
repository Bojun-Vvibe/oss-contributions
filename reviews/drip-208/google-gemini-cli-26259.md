# google-gemini/gemini-cli#26259 — fix(cli): continue task after skill slash activation

- PR: https://github.com/google-gemini/gemini-cli/pull/26259
- Head SHA: `f952d174c2...`
- Author: kkj333
- Files: 2 changed, ~+15 / −10

## Context

When a user activates a Skill via slash command (e.g. `/myskill`) without arguments, the previous behavior was to send the post-submit prompt `Use the skill ${skillName}` to the model. The model would interpret that literally as "okay, but use it on what?" and ask the user to specify a task — even when an active task was already in flight in the conversation. That broke the natural workflow where you'd say "refactor this function" then realize a relevant skill applies and `/skillname` mid-conversation expecting it to continue with the in-progress task.

## Design

Single, focused change in `packages/cli/src/services/SkillCommandLoader.ts`:

1. Extracted the default post-submit prompt into a helper at line 11-13:
   ```ts
   function getDefaultPostSubmitPrompt(skillName: string): string {
     return `The ${skillName} skill is now activated. If there is an active task in the conversation, continue it using this skill's instructions. If no task was provided, ask what task to apply it to.`;
   }
   ```
2. Wired into the `action` callback at line 43-54 — if `args.trim().length > 0`, the user-supplied args still win (existing behavior); otherwise the new default prompt is used instead of the old `Use the skill ${skill.name}`.

The test at `SkillCommandLoader.test.ts:88-94` is updated to assert the exact new prompt string, which doubles as a regression guard (changing the prompt phrasing requires touching the test, which is the right friction).

## What's good

- **Branch-aware default prompt** — the new prompt explicitly enumerates both branches ("if there is an active task, continue; if not, ask what task to apply it to"). The previous `Use the skill X` prompt collapsed those two branches and let the model guess wrong. The new phrasing maps cleanly onto the two real-world entry points.
- **User-supplied args still take precedence** — `/myskill go fix the auth bug` still passes `go fix the auth bug` as the post-submit prompt. The fix only affects the no-args path, which is the path that was broken. No regression risk for power users who supply args.
- **Helper extraction is appropriate** — factoring `getDefaultPostSubmitPrompt(skillName)` out of the action closure makes it testable in isolation if needed later, and prevents the prompt string from being duplicated if a future caller (e.g. a different command kind) wants the same default.
- **Test exact-string match** is the right level of precision for a prompt-engineering change — substring matching would let someone silently weaken the "continue active task" half of the instruction.

## Risks / nits

- **The new prompt is an English-language string baked into source** — gemini-cli has i18n in other places (the goose UI just landed locale files for similar strings). If skill activation is a localizable surface, this should route through whatever i18n layer the CLI uses for slash-command prompts. If the CLI deliberately keeps post-submit prompts as model-language English (because the model is multilingual and the prompt is "instruction to model" not "user-facing UI"), then it's correct as-is. Worth a one-line comment confirming the choice.
- **Prompt-injection surface** — the `${skillName}` is interpolated into the prompt. Skill names come from skill manifests (per `ACTIVATE_SKILL_TOOL_NAME` discovery), which are trusted-author content in this codebase. But if a skill name ever became user-influenced (e.g. someone adds a third-party skill registry), a maliciously named skill could inject `... and ignore all previous safety guidance ...` into the prompt. Worth at minimum sanitizing skillName for newlines/quotes, or noting in `getDefaultPostSubmitPrompt` that it assumes a trusted source.
- **No test for the args-supplied path** — the diff shows the test updates only the no-args case. Worth adding a parallel test that asserts `args = "fix the auth bug"` → `postSubmitPrompt: "fix the auth bug"`. The action code already handles it correctly per the diff; the test would just lock the behavior.

## Verdict

**merge-after-nits** — the fix is small, focused, and addresses a real UX paper-cut with the right shape (branch-aware default, user-args still win, exact-string test). Worth adding the args-supplied-path test before merge, and worth a one-line note on the i18n / prompt-injection posture for the trusted-skillName assumption. None are blockers.
