# sst/opencode PR #25937 — fix(server): restore web terminal CSP allowances

- URL: https://github.com/anomalyco/opencode/pull/25937
- Head SHA: `f8810e6fb1aebb729f00eb4ab0f3de2af464a875`
- Size: +54 / -15

## Summary

Restores per-HTML-body CSP computation for the embedded and proxied web UI, so that the `'sha256-…'` hash for the inline `oc-theme-preload-script` is recomputed at serve time. Previously `DEFAULT_CSP` was a hash-less constant (set at `routes/ui.ts:20` for the local-file branch and at `shared/ui.ts:60` for the embedded branch), which meant the inline preload script was blocked by the served CSP whenever the request hit those branches. Also widens `connect-src` from `*` to `* data:` so the web terminal's wasm-loaded data-URI sockets can connect.

## Specific findings

- `packages/opencode/src/server/shared/ui.ts:25-29` — new `cspForHtml(body)` helper centralizes the `themePreloadHash` → `sha256` → `csp(hash)` chain. Good consolidation; eliminates the duplicated three-line block previously at `routes/ui.ts:31-34` and `shared/ui.ts:94-95`.
- `packages/opencode/src/server/shared/ui.ts:16` — `csp()` now appends `data:` to `connect-src`. This is required for the web terminal's data-URI WebSocket bootstrap, but `connect-src * data:` is effectively unbounded — worth a one-line comment about *why* `data:` is needed so it doesn't get stripped in a later "tighten CSP" pass.
- `packages/opencode/src/server/shared/ui.ts:18` — `DEFAULT_CSP = csp()` retained for back-compat callers but the constant is now only referenced from tests. Consider deleting the export and updating the test in a follow-up.
- `packages/opencode/src/server/routes/ui.ts:19-23` — local-file branch now reads the body once into `body: Uint8Array`, decodes via `new TextDecoder().decode(body)` for the CSP computation, then returns `body` for the response. Two decodes total (one here, one inside `cspForHtml` → `themePreloadHash` regex on the decoded string) — fine for the index.html path, would be wasteful for large HTML assets, but UI fallback only serves small html files.
- `packages/opencode/src/server/routes/ui.ts:33-36` — proxy path correctly uses `await response.clone().text()` for hash computation only when `content-type` includes `text/html`; non-HTML responses get the cheap `csp()` (hash-less) variant.
- `packages/opencode/src/server/shared/ui.ts:97-101` — Effect-based `serveUIEffect` mirrors the same simplification.
- `packages/opencode/test/server/httpapi-ui.test.ts:264-294` — new test asserts (a) `script-src` contains `'wasm-unsafe-eval'`, (b) the computed sha256 of the preload script appears in the header, (c) `connect-src * data:` is present. Test is well-scoped and pins the exact regression that motivated the PR.

## Nits

- `connect-src * data:` deserves an inline comment naming the consumer (web terminal wasm socket bootstrap).
- `DEFAULT_CSP` export is now dead in production code — clean up in a follow-up rather than block this fix.

## Verdict

`merge-after-nits`
