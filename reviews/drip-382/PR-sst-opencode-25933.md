# sst/opencode PR #25933 — fix(opencode): only intercept registered local slash commands

- URL: https://github.com/anomalyco/opencode/pull/25933
- Head SHA: `25c813de3bd8e5cb28f0d7e67b2ae3eed8599150`
- Size: +66 / -0

## Summary

Closes #25932 — single-token slash inputs like `/review` were being eaten by
the local-slash fast-path in `Prompt`'s submit handler, clearing the input and
skipping the normal command dispatch even when no local TUI command of that
name existed. PR adds a registry-aware check (`command.hasSlash(name)`) and a
new `command.executeSlash(name)` helper, then routes the local-execute branch
through them so only *registered* local slash names short-circuit; unknown
slashes fall through to the existing command flow.

## Specific findings

- `packages/opencode/src/cli/cmd/tui/component/dialog-command.tsx:33-37` — new
  helper `matchesSlash(option, name)` correctly checks both the canonical
  `slash.name` and `slash.aliases?.includes(name)`, matching the existing
  `slashes()` enumerator at `:96+`. Aliases get the same first-class
  treatment as primary names. No off-by-one or case-sensitivity surprises
  (raw `===`/`.includes`, same as the surrounding code).
- `dialog-command.tsx:93-103` — `hasSlash(name)` and `executeSlash(name)`
  both gate on `isVisible(option)` before considering a candidate, so a
  command in a hidden registration tier won't be silently triggered just
  because the user typed its name. `executeSlash` returns `true` only when
  it actually invoked an option's `onSelect`, which the caller relies on.
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:831-841` —
  the new branch sits right after the early-`exit()` shortcut and before the
  `selectedModel` check. Correct placement: a registered local slash should
  bypass the "no model selected → warn" path (a `/help`-style command must
  work even when no model is configured). On match it clears
  `input.extmarks`, the input buffer, the `prompt` store fields, and the
  `extmarkToPartIndex` map — symmetric with the surrounding state-reset
  patterns elsewhere in this file. Missing a model-selection gate is
  intentional and correct.
- `packages/opencode/src/cli/cmd/tui/component/prompt/slash.ts:1-10` (new
  file) — `getLocalSlashCommand(input, hasSlash)` is a pure helper that
  trims, requires `startsWith("/")`, rejects any input containing space or
  newline (so `/review please` falls through to the normal model dispatch),
  rejects bare `/`, and finally consults `hasSlash(name)`. Correct token-
  shape for "standalone single-word slash command".
- `packages/opencode/test/cli/cmd/tui/prompt-slash.test.ts:1-26` (new file)
  — 4 cases: (a) returns name when registered, (b) returns undefined when
  unregistered, (c) returns undefined for arg / multiline forms, (d)
  returns undefined for bare `/` and plain text. Exercises every branch of
  `getLocalSlashCommand`. Notably the "registered but with arguments"
  rejection (`hasSlash` always-true, input `/spec draft`) pins the exact
  bug the issue describes.

## Notes

- New file `slash.ts` ends with no trailing newline (the diff shows
  `\ No newline at end of file`); same for `prompt-slash.test.ts`. Project
  Prettier config likely flags this — worth a `bun fmt` pass before merge.
- Helper exposes `hasSlash` and `executeSlash` as separate calls. Caller
  pattern is `hasSlash → executeSlash`, so there's a small TOCTOU window
  where a registration could be torn down between the two; in this single-
  threaded TUI it's not a real concern, but a single `tryExecuteSlash`
  returning `boolean | undefined` would be slightly cleaner. Not a blocker.
- Test file has no coverage of the alias-resolution path through the real
  `dialog-command.tsx` registry — `getLocalSlashCommand` only exercises the
  `hasSlash` callback contract. Acceptable: the alias logic lives in
  `matchesSlash` and is exercised in production by the existing
  `slashes()` consumer.

## Verdict

`merge-after-nits`
