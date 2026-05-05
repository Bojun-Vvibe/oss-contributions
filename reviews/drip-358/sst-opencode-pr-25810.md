# sst/opencode PR #25810 — fix(tui): label custom agents in dialog selector as custom or native

- Repo: `sst/opencode`
- PR: #25810
- Head SHA: `451a1d76ab02`
- Author: `WaryaWayne`
- Scope: 1 file, +1 / -1
- Verdict: **request-changes**

## What it does

In `packages/opencode/src/cli/cmd/tui/component/dialog-agent.tsx:15`, replaces the `description` field shown for non-native agents in the agent picker dialog with a hardcoded `"custom"` literal:

```diff
- description: item.native ? "native" : item.description,
+ description: item.native ? "native" : "custom",
```

So the dialog row now always shows the literal text `custom` for any non-native agent instead of the actual user-authored agent description.

## Issues

1. **Drops user-meaningful information.** `item.description` is the per-agent description that the user set (in their own agent markdown / config). For a user with several custom agents — `code-reviewer`, `release-notes-writer`, `triage-bot` — the dialog previously showed each agent's distinguishing one-liner; after this change every row reads `name | custom`, which is strictly less useful. The PR title says "label … as custom or native" but the implementation **replaces** the description field rather than **adding** a label/badge.

2. **Wrong field.** The `description` slot in this dialog rendering (see the `value/title/description` triple at `dialog-agent.tsx:13-16`) is the secondary/explanatory line for each row. The "is this native vs custom" signal belongs on a separate visual channel (a tag, a prefix `[custom]`, or a left-aligned column). Stuffing it into `description` collides with the existing UX contract and silently regresses the picker.

3. **No tests, no screenshot, no visual diff** in the PR. A 1-line TUI string swap that flips the meaning of a visible field should at minimum link to a before/after capture.

## Suggested fix

Keep both pieces of information, e.g.:

```ts
description: item.native
  ? "native"
  : (item.description ? `custom — ${item.description}` : "custom"),
```

…or, better, render the native/custom marker in a separate column / tag and leave `description` alone.

## Verdict
**request-changes** — the change as written is a UX regression for the population it claims to help (users with multiple custom agents). One-line fix to also preserve `item.description`.
