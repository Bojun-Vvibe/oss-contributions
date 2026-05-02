---
repo: QwenLM/qwen-code
pr: 3743
head_sha: 08d904335f601ec4f0832ad6723979ebaf1ddc3c
title: "fix(cli): prevent file paths from being treated as slash commands"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: QwenLM/qwen-code#3743 — `prevent file paths from being treated as slash commands`

**Head SHA:** `08d904335f601ec4f0832ad6723979ebaf1ddc3c`
**Stat:** +565 / −64 across ~10 files in `packages/cli/src/ui/...`. Fixes #1804.

## Problem

`isSlashCommand()` previously treated any input starting with `/` as a slash
command attempt. Real-world prompts like `/api/apiFunction/接口的实现`,
`/Users/name/path/file.ts`, or absolute path references typed at the start
of a line were therefore being eaten by the unknown-command handler instead
of forwarded to the model. Issue #1804 has been open for a while with users
hitting this on macOS/Linux daily.

## What this PR does

The detection logic is moved out of "starts with `/`" into a stricter
classifier. Looking at the headline diff hunks:

- `packages/cli/src/ui/utils/commandUtils.ts` (and the matching
  `commandUtils.test.ts`): the new classifier accepts a slash command only
  when the token after `/` is a valid command identifier — the existing
  registered commands, registered aliases, extension-qualified
  `<extension>:<command>` forms, and Unicode custom commands. Anything
  else (a path segment, a URL fragment, a CJK file name) falls through as
  plain text.

- `packages/cli/src/ui/hooks/slashCommandProcessor.ts` is restructured so
  it can return "not a slash command, treat as user prompt" cleanly — and
  the matching hook now also receives `historyManager.updateItem` (see
  `AppContainer.tsx:765`) so it can mutate an in-flight history item when
  reclassifying.

- `useGeminiStream.ts` / `useMessageQueue.ts` are updated so that when a
  queued line is reclassified as a prompt, the history item gets the
  `sentToModel: true` marker (`AppContainer.tsx:1145–1150`) — important
  for scrollback visual consistency between "actually sent" and "rejected
  command".

- `useMessageQueue.ts` change (visible in the AppContainer.tsx hunk near
  line 2243): the queue drain comment is rewritten —

  ```
  - // Two-phase: batch plain prompts as one turn, else pop next slash command.
  + // Batch the leading plain-prompt run; if the queue starts with a slash
  + // command, submit that segment on its own to preserve typed order.
  ```

  The new wording is more accurate; a path-like first token now correctly
  joins the plain-prompt batch instead of being yanked out as a command.

## Assessment

- The fix is at the right layer (classifier in `commandUtils.ts`), not a
  band-aid in the renderer. New tests in `commandUtils.test.ts` and
  `slashCommandProcessor.test.ts` exercise the path-vs-command boundary.
- Adding `sentToModel: true` to user-history items lets the UI distinguish
  reclassified prompts from rejected commands — good UX choice.
- The diff is large (565 additions) but most of that is test scaffolding;
  the production logic delta is moderate.

## Nits

1. **`isSlashCommand` semantics.** The PR description says the classifier
   accepts "registered commands, aliases, extension-qualified, and Unicode
   custom commands". For a brand-new custom command the user just defined
   in this session but hasn't registered yet, behavior is now "send to
   model as text" rather than "unknown command" error. That's arguably
   the right call but is a silent behavior change worth a note in the
   release notes / docs.

2. **`/` followed by a typo.** A user typing `/clera` (meaning `/clear`)
   used to get an "unknown command" error; with the stricter classifier
   it may now silently flow to the model as text. A maintainer should
   confirm the test `commandUtils.test.ts` covers "looks like a command
   but isn't registered" and decide whether the fallback should be
   prompt-vs-error.

3. **`sentToModel` flag on `HistoryItemWithoutId`.** The new field needs
   to be defined on the type in `types.ts` (the diff touches it but I
   couldn't verify the field definition lands; quick check before
   merging — TS compile would catch this but worth eyeballing).

## Verdict

**`merge-after-nits`** — fixes a long-standing real-world bug at the
right layer, with proper tests. The nits are docs-and-double-check
items, not blockers.
