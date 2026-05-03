# sst/opencode PR #25513 — feat: support serving opencode from a subpath

- **Repo:** sst/opencode
- **PR:** #25513
- **Head SHA:** `04764afdf6d8ec9b7d5a50b9c37cd631bc594dd6`
- **Author:** ramonpaolo
- **Title:** feat: support serving opencode from a subpath
- **Diff size:** +716 / -89 across ~50 files (most of which are doc translations)
- **Drip:** drip-295

## Files changed (substantive subset)

- `packages/opencode/src/server/base-path.ts` (+24/-0, new) — defines `normalizeBasePath`, `stripBasePath`, `rewriteRequestBasePath`. Handles trim, leading-slash insertion, trailing-slash strip, and `"/"` → `""` collapse.
- `packages/opencode/src/server/server.ts` (+48/-17) — threads `basePath` through `Default`, `Legacy`, `createHono`, `createHttpApi`, and a new `withBasePathHono` shim that wraps `app.fetch` / `app.request` to rewrite incoming URLs.
- `packages/opencode/src/server/routes/ui.ts` (+19/-10) — `serveUI` now strips the base path before lookup and `injectBasePath()` rewrites `</head>` to embed `<meta name="opencode-base-path" content="...">` so the SPA can read it at runtime.
- `packages/opencode/src/server/routes/instance/httpapi/server.ts` (+13/-8) — propagates `basePath` into `uiRoute`, mounts `Layer.mergeAll(...)` with the configured prefix.
- `packages/opencode/src/cli/network.ts` (+11/-1) — adds `--base-path` arg and reads `config?.server?.basePath`.
- `packages/opencode/src/config/server.ts` (+1/-0) — `basePath: Schema.optional(Schema.String)` on the server config schema.
- `packages/app/src/utils/base-path.ts` (+21/-0, new) + tests — browser-side `runtimeBasePath()`, `currentServerUrl()`, `stripBrowserBasePath()` reading the meta tag.
- 25 locale doc updates under `packages/web/src/content/docs/*` documenting the new flag.

## Specific observations

- `server/base-path.ts:1-6` — `normalizeBasePath` collapses `""`, `undefined`, and `"/"` to `""`. Good. But it does **not** validate that the base path is path-safe: `normalizeBasePath("/foo bar")` returns `"/foo bar"`, and `normalizeBasePath("/foo?x=1")` returns `"/foo?x=1"`. The injected `<meta>` content is later concatenated and consumed by URL parsing in the SPA — at minimum strip query/hash and reject whitespace, otherwise you get silent breakage when someone writes `--base-path "/opencode "` (trailing space from a copy-paste).
- `server/base-path.ts:15-22` — `rewriteRequestBasePath` constructs `new Request(url, request)` after pathname mutation. That preserves headers/method/body for fetch implementations that allow re-wrapping, but on Bun's HTTP server `Request` objects with already-consumed bodies will throw on rewrap. Worth a smoke test on POST endpoints (eg. session create) under a non-empty base path. The `httpapi/server.ts` path already does this; the test file `test/server/base-path.test.ts` only asserts GET behavior.
- `server/server.ts:567-578` — `withBasePathHono` mutates the Hono app instance by reassigning `app.fetch` and `app.request`. That's effective but it permanently shadows the originals on the captured instance, so any subsequent `Default()` consumer that treats the returned app as base-path-naïve will be surprised. Document that `Default({ basePath })` returns a base-path-aware app (no double-wrap safe).
- `server/server.ts:67-72` — the early-return shortcut `if (!opts.basePath && !opts.cors?.length)` still calls `select()` once and `DefaultHttpApi()` / `DefaultHono()` lazily; preserves the prior fast path for the common case. Good.
- `server/routes/ui.ts:438-440` — `injectBasePath` does a single `body.replace("</head>", ...)`. That works for the bundled SPA shell but is fragile: if the upstream `<head>` HTML is ever lowercased or whitespace-padded (`</head >`), nothing is injected and the SPA silently runs without a base path. Either use a regex (`/<\/head\s*>/i`) or anchor on a known marker that the SPA build emits.
- `server/routes/ui.ts:475-510` — proxied UI responses are read fully into memory (`response.clone().text()`) just to inject the meta tag. For the dev-mode UI proxy this is fine; for production SSR or large HTML this is wasteful. Acceptable for v1.
- `app/src/utils/base-path.ts:1-21` — DOM lookup is unguarded for non-browser environments (SSR, tests). The added `base-path.test.ts` mocks `document.head.innerHTML`, so jsdom is assumed. Confirm that the entry path doesn't run before `<head>` is parsed; `runtimeBasePath()` returning `""` during boot would route the SPA to wrong endpoints for a tick.
- `cli/network.ts:308-315` — precedence is `--base-path` (when explicitly set) > `config.server.basePath` > `--base-path` default. The use of `process.argv.includes("--base-path")` to detect "explicitly set" is brittle: it doesn't catch `--base-path=/foo` (single-token form). yargs/whichever parser you use exposes a `parsed.aliases` / `parsed._` map for this — prefer that.
- 25 locale doc updates are nearly identical; mechanical and low-risk. No issue.

## Verdict: `merge-after-nits`

Architecturally clean and the tests cover normalization. Three things to address before merge: (1) `normalizeBasePath` should reject query/hash/whitespace in the input, (2) `injectBasePath` regex match for case/whitespace tolerance, (3) `--base-path=/foo` detection in `cli/network.ts:308`. The Hono `app.fetch` mutation pattern works but deserves a one-line comment that the wrap is destructive.
