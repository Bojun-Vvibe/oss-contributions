---
pr: 24311
repo: sst/opencode
sha: edab9b6d657f9216ca7b3886ea08291a966bb7ca
verdict: needs-discussion
date: 2026-04-25
---

# sst/opencode#24311 — feat(app): support message annotations

- **URL**: https://github.com/sst/opencode/pull/24311
- **Author**: Kayphoon
- **Closes**: #23677

## Summary

Adds in-session message annotations: a user selects text inside a user/
assistant message, a small floating annotation trigger appears, clicking
it opens the existing comment editor, and the annotation lands in the
prompt context "basket" so the next turn can reference it. Surfaces
involved are entirely in `packages/app/`: the prompt input, the message
timeline, a new selection module, and a dedicated annotation popover/
trigger pair.

## What's actually changed

- New modules under `packages/app/src/components/prompt-input/`:
  - `message-annotations.tsx` (+248) plus `.cases.tsx` (+307) and `.test.tsx` — the rendered chip/list of annotations attached to the next prompt.
  - `action.ts` (+76) — pure helper actions for annotation manipulation, with a sibling `action.test.ts` (+164).
  - `submit.ts` gains 35 lines and `submit.test.ts` gains 425 (notable test-density bump).
  - `history.ts` extended (74+/16-) presumably to keep annotations in undo history.
- New session page modules under `packages/app/src/pages/session/`:
  - `message-annotation-popover.tsx` (+310), `message-annotation-trigger.tsx` (+278), and `message-selection.ts` (+143) — the floating UI plus selection bookkeeping.
  - `message-timeline.tsx` (+176/-1) is the only edit to existing chrome.
- Context layer: `packages/app/src/context/prompt.tsx` grows by 161+/62- and gains a 205-line test, suggesting the basket schema itself moved.
- E2E coverage: `e2e/session/session-message-annotations.spec.ts` (+541) plus selector/backend helper additions.
- `utils/message-feedback.ts` (+74) is new and paired with a 73-line test, separate from annotations themselves — worth checking the link.
- 17 i18n locale files each gain 2 lines (presumably one annotation-CTA + one accessibility label).

Overall: ~3.6k+ lines of new TS, including ~1.6k of test/e2e, with **no
backend or core protocol changes**. That is the right framing — this is
purely a frontend feature.

## Reviewable points

1. **Selection-trigger UX boundary.** The PR body explicitly says "selecting
   text no longer opens the editor immediately". That is a real behavioral
   change: existing muscle memory for users who select to copy will not
   regress, but users who relied on selection-to-comment will see a
   second click. Worth surfacing in the changelog and probably worth a
   lightweight feature-flag or settings toggle for one release so power
   users can opt back into the old "select-to-edit" behavior. The
   selection module already exists (`message-selection.ts:143`) so the
   indirection point is clean.

2. **`message-feedback.ts` colocated but separate.** A new
   `utils/message-feedback.ts` (+74) and its test (+73) are added in the
   same PR. If feedback and annotations share storage or context, that
   coupling deserves an architectural note in the PR body — otherwise
   the file looks orphaned. If they are independent, splitting the
   feedback util into a follow-up PR would shrink the review surface.

3. **i18n parity.** All 17 locale files get exactly 2 keys added. That
   is the right diligence, but `ar.ts`/`zh.ts`/etc. ship the English
   strings on day one if the team's translation pipeline is async.
   Worth a `// TODO: localize` or a fallback policy note.

4. **Test density looks honest.** `submit.test.ts` +425, `prompt.test.ts`
   +205, plus 540 lines of e2e is the right shape for a feature that
   modifies the submit pipeline. Not blocking; calling it out as a
   positive signal.

5. **No protocol/backend changes.** Annotations live in the prompt
   context basket only. That is the right scope for v1, but it means
   annotations are ephemeral per-turn and won't survive `/share` URLs
   or session export. That should be a stated non-goal in the PR body
   so reviewers and users don't expect persistence.

## Rationale

The diff is large but cleanly scoped to one package, and the test
ratio is healthy. I'm flagging this `needs-discussion` rather than
`merge-after-nits` because the selection-trigger UX change deserves a
short maintainer thread (and possibly a settings flag), and because the
unrelated-looking `message-feedback.ts` addition needs a one-line
justification in the PR body. Once those two items are addressed, this
should drop to `merge-after-nits`.

## What I learned

For frontend features that change a long-standing mouse interaction
(here: text selection → action), even a clean implementation benefits
from a short rationale block in the PR body comparing the old and new
behaviors and committing to a deprecation/transition plan. Reviewers
shouldn't have to reverse-engineer that from the diff.
