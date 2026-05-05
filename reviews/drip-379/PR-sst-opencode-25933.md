# sst/opencode #25933 — fix(opencode): only intercept registered local slash commands

- URL: https://github.com/sst/opencode/pull/25933
- Head SHA: `25c813de3bd8e5cb28f0d7e67b2ae3eed8599150`
- Author: G17hao
- Size: +66 / -0 across 4 files

## What it does

Tightens TUI prompt-submit handling so that only standalone, *registered* local slash commands are intercepted as a UI dialog action — anything else (slash-with-args, multiline, unknown name, bare `/`) falls through to normal prompt submission. Adds a `getLocalSlashCommand(input, hasSlash)` helper at `packages/opencode/src/cli/cmd/tui/component/prompt/slash.ts:1-10`, two new dialog accessors `command.hasSlash(name)` and `command.executeSlash(name)` at `dialog-command.tsx:93-103`, and wires them into the submit path at `prompt/index.tsx:831-841`. Ships a 26-line vitest covering the four interception axes (registered/unregistered/with-args/bare).

## Observations

- `slash.ts:4` short-circuits on `trimmed.includes(" ") || trimmed.includes("\n")` *after* `trimmed.startsWith("/")` — correct order, prevents `/spec something` from being silently swallowed by the dialog path. Tab-as-arg-separator is *not* covered (`"/spec\targ"` would be intercepted), but that mirrors how the existing slash-completion logic in this file treats whitespace, so consistent rather than novel.
- `matchesSlash` at `dialog-command.tsx:33-37` correctly checks both canonical `slash.name` and `slash.aliases`; `executeSlash` iterates `entries()` (not `visibleOptions()`) but re-applies the `isVisible` guard inline (`:97`), so disabled/hidden commands stay non-executable even if their name matches.
- The `executeSlash` call site at `prompt/index.tsx:832-840` clears `extmarks`, `input`, and the `prompt`/`extmarkToPartIndex` store before returning `true`. This is the right ordering — if the dialog handler later opens a modal it doesn't see stale composer state — but note the call to `option.onSelect?.(dialog)` happens *before* the clear, so any synchronous read of `store.prompt.input` inside `onSelect` still sees the `/foo` text. Worth a one-line comment.
- Test file ends with no trailing newline (`prompt-slash.test.ts:26`, `slash.ts:10`). Minor lint nit.

## Verdict: merge-after-nits

Good correctness fix for a real UX bug (typing `/spec draft` previously got eaten by the dialog system instead of submitted as a prompt). Logic is well-isolated in the new `slash.ts` helper with focused tests. Two trivial nits: missing trailing newlines on the new files, and an inline comment about the read-before-clear ordering of `onSelect(dialog)` would document the implicit contract.