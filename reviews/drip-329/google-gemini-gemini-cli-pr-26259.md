# google-gemini/gemini-cli #26259 — fix(cli): continue task after skill slash activation

- SHA: `f952d174c28c5d9fa25c391b7f5d470163b730c3`
- State: OPEN, +18/-10 across 2 files
- Files: `packages/cli/src/services/SkillCommandLoader.ts`, `packages/cli/src/services/SkillCommandLoader.test.ts`
- Related: #21165, #21760, #25318

## Summary

When a user types `/<skill-name>` with no arguments, the previous follow-up prompt was the bare `Use the skill <name>`, which is a weak signal to the model and often prompts an unhelpful "What would you like to do?" reply. PR replaces it with an explicit instruction: continue the active task using the activated skill, or ask what task if no active task exists. Explicit-args path (`/<skill-name> task description`) is unchanged.

## Notes

- `packages/cli/src/services/SkillCommandLoader.ts:11-13` — extracts the new prompt string into a top-level `getDefaultPostSubmitPrompt(skillName)` helper. Centralizes the wording so future tweaks don't sprawl. Good factoring.
- `packages/cli/src/services/SkillCommandLoader.ts:49-60` — `action` now `await`s a `const trimmedArgs = args.trim()` once and reuses it both in the truthy branch and as input to the prompt. Eliminates the previous double-`args.trim()` call. Minor perf and readability win; no behavioral change relative to the old `args.trim().length > 0 ? args.trim() : ...` pattern.
- `packages/cli/src/services/SkillCommandLoader.test.ts:88-92` — the only updated assertion. Replaces the literal `'Use the skill test-skill'` with the new explicit prompt string. No new test added for the *args-present* path, even though that path is now structurally different (single trim vs. double trim). The pre-existing test below (not shown) presumably covers it; reviewer should grep `SkillCommandLoader.test.ts` to confirm an args-present case still asserts the trimmed args are used unchanged.
- The new prompt mentions the skill name twice ("The X skill is now activated... using this skill's instructions"), which is fine — the model sees both the activation and the imperative. Worth noting that for very long skill names this prompt grows linearly, but the cap is human-typed name length, so non-issue.
- The wording "If there is an active task in the conversation, continue it" relies on the model's judgment of "active task". For brand-new conversations where the user opens with `/<skill>`, the second clause ("If no task was provided, ask what task to apply it to") correctly handles the cold-start path. The intent is sound.
- No localization concern flagged in PR — string is hardcoded English. If the project supports i18n for prompts, this should be flagged; if not, fine.
- PR body's pre-merge checklist marks docs as un-updated (`[ ] Updated relevant documentation and README (if needed)`). Skill activation behavior is user-observable; recommend a one-line docs touch wherever skill slash commands are documented.

## Verdict

`merge-after-nits` — straightforward UX improvement, correct factoring. Add a test asserting the args-present path still wins (or confirm one already exists), touch the docs page describing skill activation, and ship.
