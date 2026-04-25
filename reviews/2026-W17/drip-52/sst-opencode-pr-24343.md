---
pr: 24343
repo: sst/opencode
sha: 1d5ccbff1669daa2ccf67411f45896f521e36778
verdict: merge-after-nits
date: 2026-04-26
---

# sst/opencode#24343 — fix(session): drop stale plan reminders

- **URL**: https://github.com/sst/opencode/pull/24343
- **Author**: pascalandr
- **Files**: `packages/opencode/src/session/message-v2.ts`,
  `packages/opencode/src/session/prompt.ts`,
  `packages/opencode/test/session/message-v2.test.ts` (+52/-4)

## Summary

When a session switches out of plan mode into build/edit mode, the
synthetic `<system-reminder>Plan mode is active...</system-reminder>`
text parts that were injected into earlier user turns stay in the
history and get re-sent to the model on every subsequent turn.
Result: a build-mode agent reads "Plan mode is active. Do not edit
files." in its context and either refuses to write or asks for
confirmation. This PR adds an opt-in
`stripPlanModeReminders` flag to `toModelMessagesEffect` so any
agent *other than* the plan agent filters out those stale synthetic
reminders before serialization.

## Reviewable points

- `packages/opencode/src/session/message-v2.ts:715` — the option
  is added to both `toModelMessagesEffect` and the public
  `toModelMessages` shim (line 979). Type stays narrow:
  `stripPlanModeReminders?: boolean`. Fine.

- `message-v2.ts:774-780` — `skipText` is the predicate. Two
  important guards:
  1. `!part.synthetic` — never strips a real user-typed text
     part, even if it happens to mention "Plan mode is active".
  2. The text-match is a substring check on two literal strings,
     `"Plan mode is active"` and `"Plan mode ACTIVE"`. The second
     casing exists because the system prompt template uses both
     forms (one in the wrapper, one in the inline reminder).
     **Nit**: this is brittle — if the template ever changes the
     wording, the strip silently no-ops and the regression
     reappears. Consider tagging the synthetic part with a
     discriminator like `part.kind = "plan-mode-reminder"` and
     keying the strip on that. But for a one-line filter it's
     acceptable and the test on line 67 will fail loudly if the
     template drifts.

- `message-v2.ts:792` — the call site changes from
  `if (part.type === "text" && !part.ignored)` to
  `if (part.type === "text" && !skipText(part))`, so the existing
  `ignored` semantics still apply (skipText returns true when
  `part.ignored` is true). No behavior change for non-plan-mode
  parts.

- `packages/opencode/src/session/prompt.ts:1482` — the wiring:
  `MessageV2.toModelMessagesEffect(msgs, model,
  { stripPlanModeReminders: agent.name !== "plan" })`. This is
  the right policy — the plan agent itself still needs to see its
  own reminders in context, but every other agent (build, edit,
  custom subagents) gets a clean history.

- The test on `test/session/message-v2.test.ts:198-238` is the
  *real* contract: it asserts that `"Plan mode is active. Do not
  edit files."` is stripped while `"Your operational mode has
  changed from plan to build."` (also synthetic) is preserved.
  Good — the second message is the *transition announcement* and
  should stay; the first is the stale gate. The test makes that
  distinction explicit and would catch an over-broad regex.

## Risks

- Custom subagents named `"plan"` (a user-defined agent that
  reuses the name) would also keep the reminders. Check is purely
  on string equality. Acceptable — collisions on `"plan"` with a
  non-plan-mode agent are unlikely and the worst case is the
  current behavior (stale reminders included).
- The substring match is the brittle bit (see nit above).

## Verdict

`merge-after-nits`. Add a follow-up to attach a structured
discriminator to plan-mode synthetic parts rather than matching on
literal text — but the current PR is correct, well-tested, and
fixes a concrete user-visible regression today. Don't block on the
nit.

## What I learned

Synthetic reminder parts injected by mode wrappers are part of the
*persisted* message history and outlive the mode they were created
under. Any reminder that asserts an active operational state needs
either (a) a structured tag so it can be filtered when the state
changes, or (b) re-injection only at serialization time and never
persisted. This PR picks option (a) but does the matching on prose;
a small refactor to a discriminator field would future-proof it.
