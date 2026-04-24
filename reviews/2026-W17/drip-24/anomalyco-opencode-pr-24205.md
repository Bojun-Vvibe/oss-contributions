# anomalyco/opencode PR #24205 — fix(cli): authenticate run in-process server requests

- **Repo:** anomalyco/opencode
- **PR:** [#24205](https://github.com/anomalyco/opencode/pull/24205)
- **Head SHA:** `d7921577b637f69d830452229de6dd10a725c645`
- **Author:** rmk40 (Rafi Khardalian)
- **Size:** +58/-1 across 2 files
- **Reviewer:** Bojun (drip-24)

## Summary

Closes #24204. `opencode run` (without `--attach`) starts an
in-process server and calls it via the SDK using a custom `fetch`
that routes to `Server.Default().app.fetch(request)`. The server
applies `AuthMiddleware`, but the in-process SDK client was
constructed without auth headers. So when
`OPENCODE_SERVER_PASSWORD` is set, the in-process call fails auth,
session creation returns nothing usable, and the user sees
`Error: Session not found`.

Fix: build a `Basic` auth header from the same env vars the
server is configured with (`OPENCODE_SERVER_PASSWORD` +
`OPENCODE_SERVER_USERNAME`, defaulting username to `"opencode"`)
and pass it via the SDK `headers` option.

## Key changes

### `packages/opencode/src/cli/cmd/run.ts` (+9/-1)

Before (line 677):

```ts
const sdk = createOpencodeClient({ baseUrl: "http://opencode.internal", fetch: fetchFn })
```

After:

```ts
const headers = (() => {
  // Match the in-process server auth config; --password is only for --attach.
  const password = Flag.OPENCODE_SERVER_PASSWORD
  if (!password) return undefined
  const username = Flag.OPENCODE_SERVER_USERNAME ?? "opencode"
  const auth = `Basic ${Buffer.from(`${username}:${password}`).toString("base64")}`
  return { Authorization: auth }
})()
const sdk = createOpencodeClient({ baseUrl: "http://opencode.internal", fetch: fetchFn, headers })
```

The IIFE is reasonable here — `headers` is computed once and
passed by value. `undefined` when no password is set is the
right shape for the SDK option (no header sent at all, which is
also what the pre-PR code did, just unconditionally).

### `packages/opencode/test/cli/run.test.ts` (NEW, +49)

Spawns a child `bun run ./src/index.ts run --command
definitely-not-a-command args` with `OPENCODE_SERVER_PASSWORD=secret`
and asserts:

- output contains `'Command not found: "definitely-not-a-command"'`
  (proves command resolution was reached, i.e., session creation
  worked),
- output does NOT contain `"Session not found"` (the regression
  string).

Test runs in 15s. The use of `--conditions=browser` is curious —
worth verifying that's not masking a node-vs-browser conditional
in the auth code that would matter in production.

## Concerns

1. **`Buffer.from(...).toString("base64")` is correct for ASCII
   passwords but worth noting for non-ASCII.**

   `Basic` auth's RFC 7617 specifies UTF-8 encoding of the
   `user:pass` string before base64. `Buffer.from(string)`
   defaults to UTF-8 in Node/Bun, so this is fine. Still, an
   explicit `Buffer.from(\`${username}:${password}\`, "utf8")`
   would document the intent and survive a hypothetical future
   default change.

2. **Server-side username default not asserted to match.**

   The fix defaults username to `"opencode"` when only the
   password env var is set. If the server's `AuthMiddleware`
   uses a different default ("admin", or empty string), this
   silently still fails auth. The fix should pin this against
   the server-side default by importing it from a shared
   constant rather than hardcoding `"opencode"` on the client
   side. Otherwise a server-side default change would silently
   break this path again.

3. **Test depends on "Command not found" error text.**

   The negative assertion (`not.toContain("Session not found")`)
   is the real regression check, but the positive assertion
   (`Command not found: "definitely-not-a-command"`) couples the
   test to that exact error message. A future i18n pass or
   error-message rewording would break it. Better:
   `expect(output).toMatch(/command.*not.*found/i)` or assert
   the process exit code shape.

4. **No coverage for `OPENCODE_SERVER_USERNAME` override.**

   The test only sets `OPENCODE_SERVER_PASSWORD`, exercising the
   `??"opencode"` default branch. A second test setting both
   `OPENCODE_SERVER_USERNAME=alice` + `OPENCODE_SERVER_PASSWORD=secret`
   would pin the override path and protect against a future
   typo.

5. **`--password` flag still TODO for `run`.**

   The comment says "`--password` is only for `--attach`" — that's
   the current state, but it's a UX inconsistency. A user who
   has the server password in their password manager rather than
   their environment can pass `--password` to `run --attach` but
   not to plain `run`. Worth a follow-up to thread `--password`
   through both paths.

## Verdict

`merge-after-nits` — the fix is correct and minimal, the test
proves the regression is closed, and the IIFE is readable. Three
small polish items:

- import the server-side username default rather than
  hardcoding `"opencode"`,
- weaken the positive test assertion to a regex/exit-code check,
- file a follow-up issue for `--password` parity between `run`
  and `run --attach`.

## What I learned

This is a textbook "the server you're talking to has middleware
you forgot about" bug. The in-process SDK client + custom
`fetch` makes the auth boundary invisible — the developer reads
"in-process call" and assumes no auth needed, but
`AuthMiddleware` runs identically whether the request came over
TCP or via `app.fetch(request)`. Same class of bug as in-process
gRPC calls hitting the same interceptor stack as remote ones.
The lesson: when you build an in-process transport that
shares the server's request pipeline, every middleware in that
pipeline becomes a potential silent failure point for the
client side.
