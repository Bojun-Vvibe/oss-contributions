# block/goose PR #8881 — add skills to the chat composer

- **PR**: https://github.com/block/goose/pull/8881
- **Author**: morgmart
- **Merged**: 2026-04-28T21:35:32Z
- **Head SHA**: `662a4d99852b`
- **Size**: +2086/-540 across 49 files (UI-only, scoped to `ui/goose2/`)
- **Verdict**: `request-changes`

## Context

Skills were already discoverable in goose2's `<SkillsView>` but a user
couldn't *apply* one to a specific chat message — the skill's authored
guidance wasn't getting threaded into the agent's prompt. This PR
introduces a four-surface entry pattern (mention via `@`, slash command,
"start chat with skill" button from the Skills view, and a
chip-restoration path on session reload) that turns a selected skill
into:

1. A user-visible **chip** that renders inside the user's message
   bubble (`MessageMetadataChip` + per-bubble `messageChips` slice).
2. A separate **assistant-only prompt block** (the actual
   workflow-guidance text from the skill) that the agent sees but the
   user does *not* see in their bubble.

The split-shape is the right one — leaking the full prompt into the user
bubble would be ugly noise, but the chip is necessary so the user can
see "I sent this with skill `code-review`" when scrolling history.

## What changed (verified against diff)

- **New module** `features/skills/lib/skillChatPrompt.ts` exports
  `toChatSkillDraft(skill: SkillInfo): ChatSkillDraft`,
  `SkillCommandMatch<T>`, and the prompt-assembly helpers used by both
  the slash-command path and the mention-detection path.
- **New module** `features/chat/lib/skillSendPayload.ts` (line 926+ of
  diff) exports `buildSkillSendPayload(messageText, submittedSkills,
  ...)` and `buildSkillRetryOptions(...)`. The payload builder
  separates `messageText` (user-visible) from `sendOptions.chips` and
  the `skill_invocations: skillChips.map(c => ({ name: c.label }))`
  agent-visible payload field.
- **AppShell entry** (`AppShell.tsx:341-360`):
  `handleStartChatWithSkill(skill, projectId?)` now creates a new tab
  and seeds `useChatStore.getState().setSkillDrafts(session.id,
  [toChatSkillDraft(skill)])` before the user types anything. This is
  the "start chat with skill" surface from the Skills view.
- **Composer wiring** (`useMentionHandlers.ts`, `useChatInputSubmit.ts`,
  `useChatSessionController.ts:534`, `useMessageQueue.ts`): selected
  skills flow through composer → submit → message queue without
  losing the chip metadata. The shared `useChatInputSubmit` makes
  voice-dictation auto-submit and normal submit go through the same
  skill-aware path (good — single source of truth).
- **Render path** (`MessageBubble.tsx` ~line 2394, `MessageMetadataChip.tsx`
  ~line 2566): the chip renders in the bubble with an `IconStack2`
  glyph and `messageChipClasses[chip.type]` styling. Per-chip key is
  `${chip.type}-${chip.label}`.
- **Persistence** (`chatStore.ts` lines ~1182-1200, `draftPersistence.ts`):
  `setSkillDrafts(sessionId, skills)` and `clearSkillDrafts(sessionId)`
  are added; the draft persistence layer stores selected skills per
  session so a reload restores the chip even if the user didn't
  submit yet.
- **Replay** (`shared/api/acpSkillReplayChips.ts`, new file): replay
  path reconstructs the chips from the persisted ACP message payload
  so reloading session history shows the chips in their original
  bubbles.
- **Tests** (~12 new test files including
  `useChat.skillChips.test.ts`, `chatStore.test.ts` extension,
  `ChatInput.skills.test.tsx`, `MessageBubble.skillChips.test.tsx`,
  `SkillsView.test.tsx`): the most load-bearing one is at lines
  150-180 — `it("stores user-visible chips separately from the
  agent prompt", async () => { ... expect(message.metadata?.chips)
  .toEqual([...]) })` — which pins exactly the split-shape contract
  that this PR is built around.
- **i18n**: only `en/chat.json` and `es/chat.json` updated.

## Why `request-changes`

The shape is right and the test for the split-shape contract is
exactly the one I'd want — but four issues make me ask for changes
before this lands a wider release:

1. **Locale parity is broken.** Only `en` and `es` translation
   bundles are updated under `shared/i18n/locales/`. Goose2 ships
   with at least `de`, `fr`, `ja`, `pt-BR`, `zh-CN` (visible in the
   `locales/` directory tree). New chip-tooltip strings, the slash
   command help text, and the Skills-view "start chat with skill"
   button label will fall back to the English source string in those
   locales — which is a regression vs. the current i18n contract,
   not just an unfinished feature. Either land all locales (machine-
   translated with a TODO) or gate the new strings behind a
   feature-flag that auto-disables in unsupported locales.

2. **Assistant-only prompt block has no visible audit affordance.**
   The split — chip in the bubble, full prompt in the agent stream —
   is correct UX, but it means a user who later asks "what did I
   send?" can't see the actual prompt the agent received. There's no
   "show full prompt" expander on the chip, no debug-pane line, and
   no way for support to ask "paste the prompt that was injected by
   that chip." This is *especially* important because skills are
   user-installable and a buggy or mis-authored skill could inject
   misleading guidance the user doesn't know about. At minimum: a
   tooltip on the chip showing the first ~200 chars of the injected
   prompt; ideally: a "see full skill content" link to the skill
   detail page.

3. **`buildSkillSendPayload` skill-mapping is name-only, not
   ID-pinned.** At line 995 of diff: `skill_invocations:
   skillChips.map((chip) => ({ name: chip.label }))`. The agent
   receives only the skill *display name*, not the skill ID. If two
   skills happen to share a display name (collision across user
   skills + system skills + ACP-installed skills), the agent
   resolution is ambiguous. Even if collision is currently
   prevented by Skills-list dedup, a future Skills source could
   reintroduce it. Pin via `{ id: chip.skillId, name: chip.label }`
   and have the agent resolve by ID first, name as fallback.

4. **Replay path is silent on missing skills.** `acpSkillReplayChips.ts`
   reconstructs chips from the persisted ACP message payload, but
   if the user uninstalled a skill since the original send, the
   replay path will reconstruct a chip pointing at a skill that no
   longer exists. There's no visible test for this case in the diff
   and no obvious fallback rendering ("skill `code-review` (no
   longer installed)"). Add a test + render a muted-state chip for
   missing skills.

## Other nits (lower priority)

- The new `EMPTY_SKILL_DRAFTS: ChatSkillDraft[] = []` constant
  (diff:408) is the right pattern to avoid identity-changing array
  references in selectors, but it's declared at module scope as
  a `const` non-readonly array — a stray `.push(...)` anywhere in
  the file would mutate the singleton. `Object.freeze([])` or
  `as const` would harden it.
- `ChatInput.skills.test.tsx` and `MessageBubble.skillChips.test.tsx`
  use slightly different mock-skill shapes (visible if you grep for
  `skill-1` vs `code-review` IDs across the test files). A shared
  `tests/fixtures/skills.ts` would prevent the next test from
  drifting.
- The 49-file blast radius makes this PR hard to review per-hunk;
  a follow-up could extract `features/chat/lib/` and
  `features/skills/lib/` as a separate small "primitives" PR
  ahead of any future skill-related feature work.

## What I learned

When a UI surface has a split-shape contract — "user sees X, agent sees
X+Y, persistence stores X+Y, replay reconstructs both" — the right
test to write *first* is the one that pins the split: assert that
`message.metadata.chips` and `message.body` are computed from the same
inputs but rendered to different consumers. This PR has that test
(`useChat.skillChips.test.ts:150`) and it's the strongest piece of the
review surface. The weakest piece is exactly the *opposite* shape: when
the contract is "send a *handle* (skill ID) but display a *label*
(skill name)," the implementation collapsed handle+label down to just
label at the agent boundary, which works today but is the kind of
shortcut that bites two release cycles later. Lesson: when a UI
abstraction has a stable identifier and a mutable display string, send
both across every boundary, even if today's resolver only uses one.
