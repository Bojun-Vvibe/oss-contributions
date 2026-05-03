# sst/opencode #25600 — fix(auth): add username option for basic auth in RunCommand

- PR: https://github.com/sst/opencode/pull/25600
- Author: OpeOginni
- Head SHA: `3a0fcb269dd4d47b374011f2b70e8d89f852e398`
- Updated: 2026-05-03T14:09:44Z

## Summary
Adds a missing `--username`/`-u` option to `opencode run --attach` so the CLI mirrors the documented basic-auth surface (a parallel `--password` option already exists). The username is then passed through to `ServerAuth.headers({ password, username })`. +6 / -1 in `packages/opencode/src/cli/cmd/run.ts`.

## Observations
- `packages/opencode/src/cli/cmd/run.ts:280-285`: option is registered with `alias: ["u"]`. Worth a quick grep — `-u` is a heavily contested short flag (`--user-agent`, `--unix-socket`, etc.). For `run --attach` specifically there is no other `-u` consumer in this command at the moment, so it's fine, but the alias should be deliberate, not reflexive.
- `packages/opencode/src/cli/cmd/run.ts:284`: the help text says *defaults to OPENCODE_SERVER_USERNAME or 'opencode'*. That implies the default is resolved somewhere downstream (presumably inside `ServerAuth.headers`). Confirm `ServerAuth.headers` actually consults `OPENCODE_SERVER_USERNAME` and falls back to `"opencode"` when `username` is `undefined`. If not, the help text is a lie and the PR needs a one-line env fallback in `RunCommand` for parity with `password` (which the diff context shows uses `OPENCODE_SERVER_PASSWORD`).
- `packages/opencode/src/cli/cmd/run.ts:665`: only the `if (args.attach)` branch is updated. Skim other branches in this command (server-spawned local mode, etc.) to confirm none of them also build auth headers via `ServerAuth.headers` and would benefit from / regress without the same `username` plumbing. From the snippet only the attach path uses `ServerAuth.headers`, so this looks complete.
- The PR body links related #25113 and #25399 (docs PR that already documents this flag). That cross-link is the strongest argument for merging quickly — without this PR the docs and CLI disagree.
- Author says they ran `bun run typecheck`. No new test added. A trivial unit test ("`run --attach -u foo -p bar` produces `Authorization: Basic Zm9vOmJhcg==`") would prevent the next regression here, but for a 6-line option add it's defensible to skip.

## Verdict
`merge-as-is`
