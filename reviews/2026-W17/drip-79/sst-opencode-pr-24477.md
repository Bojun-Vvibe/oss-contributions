# sst/opencode PR #24477 — fix(tui): restore slash command text when jumping back to command messages

- **PR:** https://github.com/sst/opencode/pull/24477
- **Author:** jeevan6996 (Jeevan Mohan Pawar)
- **Head SHA:** `a4b89fc7eef45d5d5ffec842eda8ea4338b77695`
- **Files:** 6 (+101 / -51)
- **Verdict:** `merge-after-nits`

## What it does

Closes #22349. When a user message originated from a slash command or skill (e.g. `/init alpha beta`), the timeline/fork/jump-back UIs reconstructed the prompt from the *expanded template body* — a multi-kilobyte block — instead of the original `/init alpha beta` invocation. Editing that expanded blob to "tweak and resend" is unreadable.

The PR persists `command:{name,arguments,source,invocation}` metadata onto the *first text part* at command-dispatch time, then teaches three TUI dialogs (timeline, fork-from-timeline, message) to reconstruct prompts from a new shared `createPromptInfoFromParts` helper that prefers the persisted invocation over the expanded text.

## Specific reads

- `packages/opencode/src/session/prompt.ts:1622-1652` — at command dispatch, the parts list is now mapped instead of spread, and the first text part gets a `metadata.command` block:
  ```ts
  const invocation = input.arguments ? `/${input.command} ${input.arguments}` : `/${input.command}`
  const commandParts = [...templateParts, ...(input.parts ?? [])]
  const firstText = commandParts.findIndex((part) => part.type === "text")
  const parts = isSubtask
    ? [/* unchanged subtask path */]
    : commandParts.map((part, index) => {
        if (part.type !== "text" || index !== firstText) return part
        return { ...part, metadata: { ...part.metadata, command: { name, arguments, source, invocation } } }
      })
  ```
  The `findIndex` correctly skips file/etc parts so `firstText` lands on a real text part — but the empty-text-parts case is uncovered (see nit 1).
- `packages/opencode/src/cli/cmd/tui/routes/session/prompt-info.ts:1-33` (new file) — the central helper. `invocation()` defensively unpacks an `unknown` metadata bag with `typeof` guards on `command`, `invocation`, `name`, and `arguments`, falling back to `/${name}` if `arguments` is empty. Good runtime hygiene given parts come from a typed-but-evolving SDK shape.
- `packages/opencode/src/cli/cmd/tui/routes/session/prompt-info.ts:24-32` — the reduce assembles `input` from text parts *only when no command invocation was found*:
  ```ts
  if (part.type === "text" && !part.synthetic && !command) agg.input += part.text
  ```
  Subtle but correct: when a command is detected, the entire expanded body is discarded and `input` is set to `command ?? ""` in the seed. This is the core behavior fix.
- `dialog-fork-from-timeline.tsx:36-44`, `dialog-timeline.tsx:24-32` — both list builders previously did `.find(x => x.type === "text" && !x.synthetic && !x.ignored)` and rendered `part.text`. They now call `createPromptInfoFromParts(...)` and gate on `prompt.input`. Behavior parity for non-command messages is preserved because the helper accumulates text parts when no command metadata is present.
- `dialog-message.tsx:35-46` and `:72-82` — two reduce blocks were collapsed into single helper calls. Net `-22` lines, identical behavior contract.
- `test/session/prompt.test.ts:1813-1850` — new live test `command prompts tag first text part with slash invocation metadata` dispatches `prompt.command({ command: "init", arguments: "alpha beta" })`, fetches the user message, and asserts the first non-synthetic text part carries `metadata.command`. Covers the persistence half cleanly. Does *not* cover the consumer side (the three TUI dialogs).

## Nits before merge

1. **`firstText === -1` edge case in `prompt.ts:1624`** — if `commandParts` contains no text parts (theoretically possible if a command template resolves to only file parts), `findIndex` returns `-1`, the `index !== firstText` guard fails for every part (`-1` never equals a non-negative index), so no metadata is attached and the command becomes invisible to the timeline. Worth either an early return that synthesizes an invocation-only text part or a unit test that pins the contract.
2. **Helper unit tests** — `prompt-info.ts` is pure and shared by three callsites; a small unit test (no live server) covering `(a) only text parts → input is concatenated text`, `(b) text part with command metadata → input is invocation`, `(c) malformed metadata → falls through to text concat` would lock the helper before someone refactors the `unknown` unpacking.
3. **`source: cmd.source ?? "command"` fallback** — the literal `"command"` is hardcoded in `prompt.ts:1652`; if `source` is part of an enum elsewhere in the session protocol, this should reference that constant instead of a magic string.
4. **Skill invocations** — issue #22349 mentions slash commands *and* skills. The diff threads `command:` metadata; if skills dispatch through a different code path (e.g. `prompt.skill`), they may still hit the expanded-body bug. Confirm both paths are covered, or follow up.
5. **`input.arguments` whitespace** — `${input.command} ${input.arguments}` prepends a single space; multi-arg commands with leading whitespace in `arguments` produce double-spaces. Minor cosmetic.
6. **Backfill** — existing user messages in old sessions won't have `command` metadata, so the timeline still shows the expanded body for them. Acceptable for a forward-only fix; worth a one-line note in the PR body so reviewers don't expect retroactive cleanup.

Verdict: merge after the `firstText === -1` guard and a small helper unit test. Core fix is well-shaped — central helper, type-safe metadata, defensive runtime unpacking, three duplicated reduce blocks correctly collapsed.
