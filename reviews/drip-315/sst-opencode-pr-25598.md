# sst/opencode PR #25598 — feat(server): run effect-httpapi backend on Effect's BunHttpServer

- Link: https://github.com/sst/opencode/pull/25598
- Head SHA: `1ea8e8e38c1d2fcd760469e150d341285a8c9057`
- Author: kitlangton
- Size: +528 / −435 (17 files)

## Files changed (highlights)
- `packages/opencode/src/server/httpapi-listener.ts` — deleted (244 LOC PoC removed)
- `packages/opencode/test/server/httpapi-listener.test.ts` — deleted
- `packages/opencode/src/server/server.ts` — `+125 / −14` new `listenHttpApi()` path
- `packages/opencode/src/server/httpapi-server.{ts,node.ts}` — new exports gating the runtime via the `#httpapi-server` package alias
- `packages/opencode/src/server/routes/instance/httpapi/middleware/proxy.ts` — `+8 / −0` (likely upgrade plumbing)
- `packages/opencode/src/server/routes/instance/httpapi/websocket-tracker.ts` — new (55 lines)
- `packages/app/src/utils/terminal-websocket-url.{ts,test.ts}` — extracted helper + 2 unit tests
- `packages/opencode/test/server/httpapi-listen.test.ts` — new (155 lines)

## Reasoning

This is a legitimately well-motivated swap: kill the in-tree `Bun.serve({fetch})` PoC and let Effect's `BunHttpServer.layer` own the listener. The PoC's whole reason for existing — that a plain fetch handler can't surface `request.upgrade` — is correct, and `BunHttpServer.layer` is in fact the right primitive for that.

Specific things I like:
- `packages/app/src/utils/terminal-websocket-url.test.ts:5` exercises the URL-building helper directly:
  ```
  expect(url.username).toBe("")
  expect(url.password).toBe("")
  expect(url.searchParams.get("auth_token")).toBe(btoa("opencode:secret"))
  ```
  That test is the right shape — it pins down that creds are *not* embedded in the WS URL, which was the actual security-relevant change in `terminal.tsx`.
- `packages/app/src/components/terminal.tsx:474` switches from a `NotFoundError`-name sniff to checking `result.response.status === 404`:
  ```
  .get({ ptyID: id }, { throwOnError: false })
  .then((result) => result.response.status === 404)
  ```
  Cleaner and removes a string-equality check that broke when the underlying SDK changed error names.

Concerns to surface before merge:

1. **Conditional package export points only to the node file.** `packages/opencode/package.json` declares:
   ```
   "#httpapi-server": {
     "bun": "./src/server/httpapi-server.node.ts",
     "node": "./src/server/httpapi-server.node.ts",
     "default": "./src/server/httpapi-server.node.ts"
   }
   ```
   All three conditions resolve to the same `.node.ts` file. Either the bun-specific module hasn't been written yet, or the `bun:` key is dead weight. Given the PR's whole point is to use `BunHttpServer.layer`, the intent is presumably to add a `httpapi-server.bun.ts` later — leaving this asymmetric in `package.json` invites the wrong runtime being picked up under bundlers. Worth at least a TODO comment.
2. **`R = any` cast at the layer boundary** is acknowledged in the PR description ("`HttpMiddleware` returns `Effect<…, any, any>` by interface design"). That's defensible but should land with a comment in `server.ts` explaining why the cast is sound, so future reviewers don't try to "fix" it.
3. **PR description checks two manual test plan items as unchecked** ("Manual: opencode serve with HttpApi backend; exercise PTY + workspace-proxy WS through the SDK", "Followup: Node parity"). Worth running the manual exercise before merge — the whole point is upgrade-handling correctness, which the unit tests don't cover.

## Verdict

`merge-after-nits`

Net negative LOC, replaces hand-rolled WS upgrade plumbing with the proper Effect primitive, adds focused unit tests for the URL builder and a 155-line listener integration test. Address the `package.json` symmetry / TODO and run the manual PTY+WS smoke before merging.
