# Review: google-gemini/gemini-cli #26431

- **Title:** fix(cli)#21297: clear skills consent dialog before reload
- **Head SHA:** `78db2d93bb0427145a5d2c631da82001cf2a8b0b`
- **Scope:** +67 / -2 across 5 files
- **Drip:** drip-339

## What changed

Fixes #21297 where the skills consent dialog re-appeared after the user
accepted because the consent state was not cleared before the extension
reload triggered a re-render. Adds focused unit tests for both the
consent state and the `/skills` command.

## Specific observations

- `packages/cli/src/config/extensions/consent.ts` (+5 / -1): the actual
  fix — looks like the consent record is now cleared/set before the
  reload signal fires. Verify the new code path runs synchronously
  *before* the reload, otherwise the race the bug describes can still
  occur on slow disks.
- `packages/cli/src/config/extensions/consent.test.ts` (+29 / -0): good
  regression test. Confirm it asserts the *order* of operations
  (clear-then-reload), not just the end state, since end-state alone
  doesn't catch the race.
- `packages/cli/src/ui/commands/skillsCommand.ts` (+1 / -0): single-line
  change. Likely wires the cleared consent into the command handler;
  confirm there is no orphan stale closure capturing the old consent
  reference.
- `packages/cli/src/ui/commands/skillsCommand.test.ts` (+31 / -0): new
  test file mirrors the consent-test additions. Good coverage parity.
- `packages/cli/src/ui/commands/types.ts` (+1 / -1): a single-char/type
  signature tweak; likely widens the command return type to include the
  new clear action. Trivial.

## Risks

- Race between dialog dismissal and reload still possible if the new
  clear is async.
- Otherwise tightly-scoped bugfix with paired unit tests.

## Verdict

**merge-after-nits** — confirm synchronous ordering of clear→reload in
the test, then ship.
