---
pr: google-gemini/gemini-cli#26229
sha: 8804e3031e74ad08892bf832a112550fb9ab63d9
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(ui): made shell tool header wrap on Ctrl+O

URL: https://github.com/google-gemini/gemini-cli/pull/26229
Files: `packages/cli/src/ui/components/messages/{ShellToolMessage.tsx,ToolMessage.tsx,ToolShared.{ts,test.tsx},ShellToolMessage.test.tsx}`

## Context

`ShellToolMessage` and the generic `ToolMessage` render a one-line header
with `wrap="truncate"`. Long shell commands (or long tool descriptions)
got cut off mid-string, and there was no way for the user to expand the
header to read the full text. Ctrl+O already toggles per-tool expansion
state via `ToolActionsContext.isExpanded(callId)`, but neither header
component consulted it.

## What changed

In `ShellToolMessage.tsx:62-65` and `ToolMessage.tsx:68-71` (parallel
edits), the header now derives:

```ts
const isExpanded =
  (isExpandedInContext ? isExpandedInContext(callId) : false) ||
  availableTerminalHeight === undefined;
```

and forwards `isExpanded` to `<ToolInfo isExpanded={...}>` (lines 180,
113 respectively). `ToolInfo` (in `ToolShared.ts`, not in this diff
excerpt but exercised by the new test) presumably switches its `wrap=`
prop based on the flag. The `availableTerminalHeight === undefined`
fallback preserves the existing behavior for non-pty render paths
(static history, snapshot tests) where there's no height budget to
respect.

Tests at `ShellToolMessage.test.tsx:359-415` cover three states:
truncated by default with finite height, expanded when height is
undefined, expanded when context returns `true` for the call id. A
sibling test in `ToolShared.test.tsx:69-99` exercises `ToolInfo`
directly with and without `isExpanded`.

## Nits

- The two parallel `isExpanded` derivations in `ShellToolMessage.tsx:62`
  and `ToolMessage.tsx:68` are byte-identical. Worth lifting into a
  `useToolHeaderExpansion(callId, availableTerminalHeight)` hook to
  prevent drift — if a third tool component needs the same behavior
  next quarter, the temptation to copy a third time is real.
- `isExpandedInContext ? isExpandedInContext(callId) : false`: the
  context's `isExpanded` is typed as optional. Once typed as required-
  but-defaulting-to-`() => false` in the provider, this guard goes
  away. Minor, safe to defer.
- The "Header Expansion" test block at `ShellToolMessage.test.tsx:359`
  asserts `lines.length > 5` for the expanded cases, which is brittle
  (depends on terminal width × description length math). A more
  targeted assertion — that the wrap mode prop changed, or that the
  full description string appears in `lastFrame()` — would survive
  layout tweaks better.

## Verdict

`merge-after-nits` — the user-visible behavior is correct and the test
coverage is real, but the duplicated derivation across two components and
the brittle line-count assertion are worth a follow-up before this
pattern spreads further.
