# sst/opencode #24829 — fix(web): stop credential prompt loop with text/plain 401 + manifest credentials

- PR: https://github.com/sst/opencode/pull/24829
- Head SHA: `fcd9e34cd74530f4ea33604dbd4b087ecf7c7831`
- Files: `packages/app/index.html`, `packages/opencode/src/server/middleware.ts`, `packages/opencode/test/server/auth-middleware.test.ts` (new, 102 lines)

## Citations

- `packages/app/index.html:11` — `<link rel="manifest" href="/site.webmanifest" crossorigin="use-credentials">` (added attribute). The inline comment says: "required so the manifest fetch carries the page's basic-auth credentials when OPENCODE_SERVER_PASSWORD is set; same-origin so no CORS preflight."
- `packages/opencode/src/server/middleware.ts:39` — `AuthMiddleware` switches from sync `MiddlewareHandler` to `async`, then wraps the `basicAuth({ username, password })(c, next)` call in try/catch.
- `middleware.ts:56-65` — the catch checks `err instanceof HTTPException && err.res && err.res.status === 401 && !err.res.headers.get("content-type")` and rebuilds a 401 with `text/plain; charset=utf-8` while preserving headers (notably `WWW-Authenticate`) and `err.res.body`.
- `test/server/auth-middleware.test.ts:38` — six-test regression suite asserts: missing creds → text/plain & no octet-stream; wrong creds → text/plain; correct creds → 200 passthrough; no password → 200 passthrough; OPTIONS preflight bypasses; `auth_token` query param works.

## Verdict

`merge-as-is`

## Reasoning

Two independent root causes for the same user-visible symptom (basic-auth dialog flashing 2-3 times, Playwright surfacing `Error: Download is starting`), and the PR fixes both with surgical, narrowly-scoped changes. The diagnosis is *exactly* the kind of thing that takes a long debugging session to pin down and is easy to validate once written down.

Cause #1 — `hono/basicAuth` builds its 401 with only `WWW-Authenticate` set and no `Content-Type`. Bun's HTTP serializer fills that slot with `application/octet-stream`. Chromium's response sniffer treats `application/octet-stream` as a download attachment for a brief window before the auth headers cause it to fall back to the basic-auth dialog. That race is what produces the flashes and the Playwright "Download is starting" symptom. The fix walks straight up the failure mode: catch hono's HTTPException at the boundary, copy the headers, set `content-type: text/plain; charset=utf-8`, and re-throw. Critically the body is preserved (`err.res.body`) and so is `WWW-Authenticate` (the `new Headers(err.res.headers)` clone), so RFC-7235 behavior is intact.

Cause #2 — same-origin manifest fetches don't carry the page's credentials by default. Browsers issue an *independent* basic-auth dialog for `/site.webmanifest` because the fetch is treated as anonymous. `crossorigin="use-credentials"` on a same-origin URL forces credential propagation without triggering CORS preflight (it would on cross-origin). The HTML comment is exactly right and worth keeping in the source — this is the kind of single-attribute fix that drifts back out a year later in a "cleanup unused attribute" PR.

The defensiveness in the catch block is well-judged: only rewrites when status is 401 *and* content-type is missing — won't double-wrap if hono ever fixes this upstream, won't touch other HTTPExceptions. The conjunction is tight in the right way.

The test file is the model shape for this kind of fix: 6 tests, each one pinning a specific behavior, with the regression-doc comment explaining *why* the test exists ("Bun's HTTP serializer then defaults to application/octet-stream..."). The OPTIONS preflight test (`expect(response.status).not.toBe(401)`) is the right shape — it asserts the auth bypass happened without overcommitting to the 404-vs-405 result. The `auth_token` query-param test catches a regression in the parallel pre-existing code path (lines 47, `c.req.raw.headers.set("authorization", ...)`).

This ships.
