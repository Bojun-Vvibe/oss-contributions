# sst/opencode PR #25397 — feat(opencode): add username option for basic auth in attach command

- URL: https://github.com/sst/opencode/pull/25397
- Head SHA: `f2d030764563b7f9e7270d04e0b4c64f23e50412`
- Closes: #25113, #18611
- Verdict: **merge-as-is**

## What it does

Adds a `--username / -u` flag (and corresponding env var fallback) to two
commands that previously hard-coded the basic-auth username as `opencode`:

- `packages/opencode/src/cli/cmd/run.ts` (the `RunCommand`'s `--attach` path)
- `packages/opencode/src/cli/cmd/tui/attach.ts` (the standalone `attach`
  command)

The resolution order is `args.username ?? process.env.OPENCODE_SERVER_USERNAME
?? "opencode"`, which exactly mirrors the existing `--password` resolution.

## Specifics I checked

- `run.ts:269-274` adds the yargs `.option("username", { alias: ["u"], ... })`
  next to the existing `--password` option. Good consistency.
- `run.ts:660` updates the inline header builder to consume `args.username`
  with the same fallback chain — the previous behavior (env then `"opencode"`)
  is fully preserved when `--username` isn't passed. No breaking change.
- `tui/attach.ts:38-46` mirrors the new option on the attach command.
- `tui/attach.ts:71-72` constructs `Basic <base64(user:pass)>` correctly.
  Buffer is fine in Node; this isn't running in browser context.
- Docs are updated in `cli.mdx` for the `attach` and `run` flag tables across
  all locales — no doc drift.

## What I like

- The change is minimal and additive. Old scripts that relied on
  `OPENCODE_SERVER_USERNAME` (or the `opencode` default) keep working
  byte-identically.
- `-u` is a conventional short flag for username and doesn't collide with
  existing run/attach short flags (`-c`, `-s`, `-m`, `-f`, `-p`).

## Nits (non-blocking)

- No test exercising the resolution precedence (`args` > env > default). It's
  three lines per command and would prevent future "I swapped the order"
  regressions, but the original `--password` path is also untested — fine to
  defer.
- Could also accept `username:password` in the password arg as an
  alternative encoding, but that's out of scope.

## Risk

None. Pure additive flag with safe default.
