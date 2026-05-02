# sst/opencode PR #25410 — fix(cli): strip leading slash from --command argument

- Repo: `sst/opencode`
- PR: #25410
- Head SHA: `b2221a093fbad12291ffff3bc5074552bc83f531`
- Author: 21pounder

## Summary
`opencode run --command /test` previously failed with "Command not found"
because the registry stores names without the leading slash. The fix strips
a single leading `/` before dispatch in `packages/opencode/src/cli/cmd/run.ts`
and updates the option `describe` text. Closes #25364.

## Specific references
- `packages/opencode/src/cli/cmd/run.ts:217` — describe text now reads
  "the command to run (without leading slash), use message for args". Good
  inline expectation-setting.
- `packages/opencode/src/cli/cmd/run.ts:635` — `command:
  args.command.replace(/^\//, "")` strips exactly one leading slash. Regex
  is anchored with `^`, so internal slashes (e.g. `/foo/bar`) are preserved
  as `foo/bar` — matches the registry key shape the PR description traces
  through `prompt.ts:1521`.

## Verdict
`merge-after-nits`

## Rationale
Two-line fix matching a real UX papercut, with the trace through
`commands.get(input.command)` documented in the PR body. Two nits:
1. `args.command` is typed as `string` per the yargs `.option("command",
   { type: "string" })` definition, but yargs returns `undefined` when the
   flag is omitted. The current call site is inside a branch that already
   checked `args.command` truthiness upstream — worth a quick re-check that
   line 635 isn't reachable with `args.command === undefined`, otherwise
   `.replace` will throw `TypeError`. A `args.command?.replace(/^\//, "")`
   would be defensively safer for ~zero cost.
2. Consider also stripping a trailing space (`/test ` from a fat-fingered
   shell paste) — but that's strictly a nice-to-have; the reported issue
   is just the leading `/`.

No tests added; the change is small enough that a single regression test
covering both `test` and `/test` going through `RunCommand` would have
been ideal but is not blocking.
