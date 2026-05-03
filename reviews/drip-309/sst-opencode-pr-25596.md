# sst/opencode #25596 — fix(auth): respect server username in clients

- PR: https://github.com/sst/opencode/pull/25596
- Author: nexxeln
- Head SHA: `5d5bd389eb9e9c1163af84e1dd342da6067176e7`
- Updated: 2026-05-03T13:48:25Z

## Summary
Extracts the duplicated Basic-auth header construction (`OPENCODE_SERVER_USERNAME` + `OPENCODE_SERVER_PASSWORD`) into a single `ServerAuth` module under `packages/opencode/src/server/auth.ts` and threads it through `acp`, `run --attach`, `tui attach`, `tui worker`, and the plugin client. Also adds a `--username/-u` flag to `tui attach`, fixing a real bug where that command hardcoded `opencode` as the username and ignored both env var and explicit credentials.

## Observations
- `packages/opencode/src/server/auth.ts:1-48` (new file): clean public surface — `Config`, `required`, `authorized`, `header`, `headers`. The `Config` service is renamed from `@opencode/ExperimentalHttpApiServerAuthConfig` to `@opencode/ServerAuthConfig`. That string is the Effect Context tag, so any external consumer keying off the old tag (other workspace plugins, downstream forks) will silently break — flag this in the PR description and/or keep the old tag as an alias.
- `packages/opencode/src/cli/cmd/tui/attach.ts:38-46`: the new `--username/-u` flag is the actual user-visible bugfix, but the `describe` says `defaults to OPENCODE_SERVER_USERNAME or 'opencode'` while the implementation in `auth.ts:199` uses `credentials?.username ?? Flag.OPENCODE_SERVER_USERNAME ?? "opencode"`. That matches — good. But `args.username` from yargs is `string | undefined`, and yargs treats `-u ""` as an empty string, not undefined; `ServerAuth.headers({ username: "" })` would then fall through to env/default because of `??`, which is fine. No action required, just confirm intentional.
- `packages/opencode/src/cli/cmd/tui/worker.ts:110-117`: `getAuthorizationHeader()` was deleted and replaced inline with `ServerAuth.header()`. Good. Note though that worker.ts uses `btoa` (browser/edge runtime) where the new `header()` uses `Buffer.from(...).toString("base64")` (Node). For ASCII passwords this is equivalent, but for UTF-8 passwords containing non-Latin1 bytes the encodings differ (`btoa` throws on non-Latin1, `Buffer.from` does UTF-8). This is actually a quiet bugfix-within-a-bugfix; worth a one-line note in the PR body so reviewers know `worker.ts` semantics changed.
- `packages/opencode/src/server/routes/instance/httpapi/middleware/authorization.ts:222-261` (deleted block): the old `isAuthRequired` and `isCredentialAuthorized` are gone, replaced by `ServerAuth.required` / `ServerAuth.authorized`. Logic is byte-identical — verified at `auth.ts:183-193`. Safe move.
- `packages/opencode/test/server/auth.test.ts:355-407` (new): mutates `Flag.OPENCODE_SERVER_PASSWORD` and `OPENCODE_SERVER_USERNAME` directly. If `Flag` is shared module-level mutable state (it appears so), tests in the same bun process running in parallel could race. The `afterEach` restore is good but test isolation depends on bun's file-level serial execution. Confirm the bun test runner config does not run files in parallel within the same module.
- No CHANGELOG entry / changeset update visible. For a user-visible CLI flag (`-u`) and a documented env var change, a changeset is warranted.

## Verdict
`merge-after-nits`
