---
pr: 24477
repo: sst/opencode
sha: a4b89fc7eef45d5d5ffec842eda8ea4338b77695
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24477 — fix(tui): restore slash command text when jumping back to command messages

- **Head SHA**: `a4b89fc7eef45d5d5ffec842eda8ea4338b77695`
- **Author**: jeevan6996
- **Size**: medium (new `prompt-info.ts` factory + 3 dialog-call-site refactors + `session/prompt.ts` part-emission tweak)

## Summary
Centralizes `parts → PromptInfo` reconstruction in a new `routes/session/prompt-info.ts` so dialog-message, dialog-timeline, and dialog-fork-from-timeline all use the same logic. The factory recognizes a part with `metadata.command` and reconstructs the original `/command args` invocation as the input field, instead of falling through to the rendered template text.

## Specific findings
- `packages/opencode/src/cli/cmd/tui/routes/session/prompt-info.ts:5-22` — `invocation()` walks `part.metadata.command` and prefers `command.invocation` (string), falling back to `/${name} ${arguments}`. Good shape, but the type chain is `(metadata as Record<string, unknown>)["command"]` then re-cast and re-cast — easy to drift if the part metadata schema gains another field. Worth a tiny zod schema or at minimum a typed helper `getCommandMetadata(part: Part): { invocation?: string; name?: string; arguments?: string } | undefined` so the three string field accesses share one validator.
- `packages/opencode/src/cli/cmd/tui/routes/session/prompt-info.ts:24-32` — `createPromptInfoFromParts()` does `parts.reduce<string | undefined>(...)` to find *the first* command invocation, then in the second reduce: `if (part.type === "text" && !part.synthetic && !command) agg.input += part.text`. Two subtle issues:
  1. If a single message contains both a command part *and* free-form user text, the free text is silently dropped. Concretely: user types `/explain extra context here` — the `arguments` field captures `extra context here`, but if the command renders a template that includes a separate text part, that template text is intentionally dropped (good). However, if the user later edits and adds a *trailing* text part not associated with the command, it disappears. Worth confirming that path isn't reachable through the timeline / fork dialogs.
  2. `agg.input = command ?? ""` at the end overrides any text accumulated during reduce when `command` is set, but the conditional inside the reduce already guards (`!command`). Belt-and-suspenders, but the dual write is confusing — pick one.
- `packages/opencode/src/cli/cmd/tui/routes/session/dialog-fork-from-timeline.tsx:36-44` — old code skipped messages with no non-synthetic text part; new code skips messages with no `prompt.input`. For a message that is *only* a command (template-text + file parts but no user text), old behavior was "skip", new is "show as `/command args`". That's the bug fix. Confirm this is intended for the *fork-from-timeline* selector — fork-from-command is now possible, where it wasn't before. Probably correct.
- `packages/opencode/src/cli/cmd/tui/routes/session/dialog-timeline.tsx:24-32` — same inversion as above, applied to the read-only timeline browser. Correct and symmetric.
- `packages/opencode/src/cli/cmd/tui/routes/session/dialog-message.tsx:35-40, 72-76` — both `setPrompt` and the navigate-after-fork path now use `createPromptInfoFromParts(sync.data.part[msg.id] ?? [])`. The `?? []` matters — old code did `sync.data.part[msg.id].reduce(...)` which would crash if the part array hadn't loaded yet. Latent fix worth calling out.
- `packages/opencode/src/session/prompt.ts:1622-1655` — emit-side change: `commandParts.map((part, index) => { if (part.type !== "text" || index !== firstText) return part; ... })` rewrites the first text part of a command invocation to attach `metadata.command = { invocation, name, arguments }`. This is the producer half of the contract that `prompt-info.ts` consumes. Looks correct, but the diff is truncated at the closing brace so I can't verify the full rewrite shape — make sure `templateParts` for subtask agents (the `isSubtask` branch) doesn't lose the metadata, since the subtask path goes through a different prompt-text injection.
- No new tests for `createPromptInfoFromParts`. Given it's now the single source of truth across three dialogs, a unit test with three fixtures (command-only, command+files, plain-text-message) would lock the contract.

## Risk
Low to medium. Touches the prompt-rebuild path that the entire timeline/fork UX depends on; bug regression here would be very visible.

## Verdict
**merge-after-nits** — add a zod-validated metadata helper, drop one of the redundant `command` writes in the reducer, and ship a small unit test for `createPromptInfoFromParts`.
