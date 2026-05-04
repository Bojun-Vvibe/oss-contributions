# Review: google-gemini/gemini-cli #26428

- **Title:** fix(cli)#22185: add IDE paste bindings to terminal setup
- **Head SHA:** `b9f7c455e7fd4d892dbb47a0b89c67b669e373c9`
- **Scope:** +141 / -12 across 7 files
- **Drip:** drip-338

## What changed

Extends `/terminal-setup` so it injects three additional VS Code-style
keybindings (`ctrl+v`, `cmd+v`, `alt+v`) that route paste through
`workbench.action.terminal.sendSequence` with kitty-keyboard-protocol
escape sequences (`[118;9u`, `[118;3u`, `\u0016`). Also widens the
supported-IDE list from "VS Code, Cursor, Windsurf" to include
"Antigravity", and bumps the binding-array length expectation from 6 to
9 across the snapshot/test suite.

## Specific observations

- `packages/cli/src/ui/utils/terminalSetup.ts` (diff truncated past the
  first 200 lines) — the binding payloads use the kitty CSI-u format
  for `cmd+v` / `alt+v` and the literal SYN (`\u0016`) for `ctrl+v`.
  That asymmetry is correct: the CLI's input layer reads CSI-u for the
  cmd/alt modifier combos but `ctrl+v` already arrives as SYN on most
  terminals, so re-emitting that byte preserves the existing handler.
  Confirm the asymmetry is documented near the binding constants.
- `packages/cli/src/ui/utils/terminalSetup.test.ts:137-204` — the new
  "should upgrade existing multiline-only setup with paste bindings"
  test covers the migration path: an existing 6-binding VS Code config
  is read, the three paste bindings are added, and the final length
  reaches 9. Asserts on `expect.arrayContaining([…])` so binding
  ordering isn't over-specified — good.
- `packages/cli/src/ui/utils/terminalSetup.test.ts:214-230` — the
  "should not modify if bindings already exist" test was updated to
  include the three new entries, otherwise it would have flagged the
  upgrade path as needing a write. Necessary update, easy to miss.
- `packages/cli/src/ui/utils/__snapshots__/terminalSetup.test.ts.snap`
  diff adds the three paste-binding objects in the expected slot order.
  Snapshot test is the one place where ordering is locked in — verify
  the implementation also writes them in that exact order so the
  snapshot doesn't oscillate.
- `packages/cli/src/ui/commands/terminalSetupCommand.ts:9-22` — the
  command description string is updated to "multiline input and paste
  (VS Code, Cursor, Windsurf, Antigravity)". The test on line 22 of
  `terminalSetupCommand.test.ts` asserts both `'paste'` and
  `'Antigravity'` substrings are present. Good explicit coverage.
- `docs/reference/commands.md:453` and
  `packages/cli/src/ui/constants/tips.ts:158` — docs and tip strings
  updated in lockstep with the command description. No stale references
  left behind.

## Risks

- The `\u0016` (`^V`) literal can collide with a user's existing VS Code
  binding for paste-as-quoted-input. Worth a one-line note in the
  release notes that `/terminal-setup` will overwrite any existing
  `ctrl+v`/`cmd+v`/`alt+v` terminal bindings in the IDE config.
- Antigravity is an actively-developing IDE; binding format compatibility
  should be re-verified each release. Not blocking.

## Verdict

**merge-after-nits** — fix is well-tested with both happy-path and
upgrade-path coverage; ask for a brief inline comment on the
ctrl+v vs. cmd+v/alt+v escape-sequence asymmetry near the binding
constants.
