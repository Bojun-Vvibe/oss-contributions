# google-gemini/gemini-cli PR #25801 — fix `/clear (new)` slash-command name

- **PR**: https://github.com/google-gemini/gemini-cli/pull/25801
- **Author**: @mini2s
- **Head SHA**: `6ac51bda94b636084ec6dbab0c57537a16239a28`
- **Size**: +1 / −1
- **Files**: `packages/cli/src/ui/commands/clearCommand.ts`

## Summary

The built-in slash command was registered with `name: 'clear (new)'` and `altNames: ['new']`. That means typing `/clear` would not match (the literal name is `clear (new)` with a space and parens), forcing users onto the alias `/new` or the literal `/clear (new)`. Fix: rename the canonical name to `clear` so the documented `/clear` works.

## Verdict: `merge-as-is`

Trivially correct. The literal `'clear (new)'` was almost certainly a placeholder from a "new clear behavior" rollout that someone forgot to clean up; `altNames: ['new']` already provided the `/new` alias, so no behavioral regression is possible. After the fix the command surface is `name: 'clear'` + `altNames: ['new']`, which matches user expectation, command-palette autocomplete, and (almost certainly) the docs.

## Specific references

- `packages/cli/src/ui/commands/clearCommand.ts:18-19` — single-line change:
  ```diff
  -  name: 'clear (new)',
  +  name: 'clear',
     altNames: ['new'],
  ```

## Nits / things to verify before merge

1. **Doc/test grep.** Search the repo for the literal string `clear (new)` (with the space and parens) — if any test, snapshot, integration test, or user-facing doc has been asserting on `'clear (new)'`, it'll need a matching update. Likely none, but the diff being a one-liner argues for a defensive `git grep` rather than a "looks fine".
2. **Telemetry / logs.** If command-name strings are emitted as telemetry dimensions, the cardinality of `slash_command_name` will shift from `clear (new)` to `clear`. Usually desirable (the new value is the "real" name), but worth noting in the PR body.
3. **Snapshot tests.** UI snapshot tests of the help / command-palette listing may include `clear (new)`; CI will catch this but the author should re-run snapshots locally.

## What I learned

This is the textbook "shipped placeholder" bug: a developer working on a v2 of an existing command appended `(new)` to disambiguate during dev, the v1 got removed, and the v2's name kept the parenthetical. The single most useful guardrail against this class is a unit test that asserts `every SlashCommand.name matches /^[a-z][a-z0-9-]*$/` — a one-line invariant test would have caught this on the original landing PR. Worth proposing as a follow-up.
