# anomalyco/opencode#24241 — fix(tui): clean zero-width agent display labels

- PR: https://github.com/anomalyco/opencode/pull/24241
- Author: Ethan0x0000
- +422 / -31
- Base SHA: `cdc7d5f2eafa62158a6305025d4c1b37d961c40f`
- Head SHA: `94475a2a9bac63073ab7515f1fd2a6a39914d951`
- State: OPEN

## Summary

Adds a centralized `AgentDisplay` helper module
(`packages/opencode/src/agent/display.ts`) that strips leading
zero-width sort prefixes (`U+200B`, `U+200C`, `U+200D`, `U+FEFF`)
from agent names for display purposes, and refactors the agent
dialog and prompt-autocomplete components to route every
agent-name render through it. The motivation: users (or agent
templates) have been prepending zero-width characters to control
sort order in the TUI agent picker, but those characters then
leak into mention chips, dialog titles, and autocomplete
suggestions as invisible-but-clipboard-poisoning artifacts.

## Specific findings

- `packages/opencode/src/agent/display.ts:1` (head SHA
  `94475a2a`) — `LEADING_INVISIBLE_SORT_PREFIX =
  /^[\u200B\u200C\u200D\uFEFF]+/`. Anchored to start (`^`) only,
  which is exactly right: trailing or interior zero-width chars
  may be intentional (combining-form fixes for non-Latin scripts);
  only the *leading sort prefix* is the documented anti-pattern
  this is trying to clean.
- `agent/display.ts:4` — the `displayName` fallback `next.length
  === 0 ? name : next` correctly handles the pathological case
  where the agent's whole name is zero-width characters (some
  joke agent named entirely in `U+200B`s). Better to render the
  raw garbage than an empty string that the layout engine will
  collapse to a zero-width slot.
- `dialog-agent.tsx:18` — `agentDialogTitle` is a thin wrapper that
  exists primarily so the dialog has a documented seam if it
  ever needs different display rules from mentions. Slight
  over-abstraction (the wrapper has no behavior beyond
  `AgentDisplay.displayName`), but cheap and the JSDoc value is
  real.
- `prompt/autocomplete.tsx:71` — `agentAutocompleteAliases`
  returns `[...new Set([\`@${agentName}\`,
  agentAutocompleteDisplay(agentName)])]`. This preserves the
  original (zero-width-prefixed) form as a typed alias so users
  who got muscle memory typing `@​agent` (with the prefix) still
  match — but the *displayed* completion is the cleaned form.
  This is the correct UX trade-off: clean what the user sees,
  loose-match what the user types.
- `agentAutocompleteOption` at `autocomplete.tsx:81` inserts
  `agent.name` (the *raw*, prefixed name) into the prompt parts.
  This is intentional: the persisted message reference must
  match the agent's canonical identifier in the agent registry,
  not its cleaned display form. Display vs identity separation
  is correctly maintained.

## Risks

- The new `agent/display.ts` module is consumed by two files in
  this diff (`dialog-agent.tsx`, `autocomplete.tsx`) but the
  `+422` line count suggests there are more touch points. Any
  call site that renders an agent name and *isn't* migrated will
  visually disagree with the migrated ones. A grep for raw
  `agent.name` in JSX after merge would catch leakage.
- No test file for `display.ts` is visible in the diff slice
  reviewed. A 5-line vitest covering the four prefix codepoints
  + the all-zero-width fallback would lock in the regex shape.

## Verdict

`merge-after-nits`

## Rationale

The shape is right (centralize the cleaner, route all display
sites through it, preserve raw identity for persistence and
matching) and the regex is conservatively scoped to the leading
sort-prefix abuse case. Add a unit test for `displayName` and
audit the remaining `agent.name` JSX call sites before merge.

## What I learned

Zero-width sort prefixes are a real anti-pattern in any UI that
exposes a user-controlled label as both a sortable list entry
and a clipboard-copyable identifier. The fix is always:
display-clean, identity-raw, match-loose.
