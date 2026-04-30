---
pr-url: https://github.com/sst/opencode/pull/25099
sha: 62e1335388fd
verdict: merge-after-nits
---

# fix(opencode): allow oc://renderer origin in CORS middleware

One-line addition at `packages/opencode/src/server/middleware.ts:77`: `if (input.startsWith("oc://renderer")) return input` slotted between the existing `http://localhost:` / `http://127.0.0.1:` allow-list arms and the `tauri://localhost` family arm. Reflecting an `Origin` header back as the `Access-Control-Allow-Origin` value (which is what the surrounding `CorsMiddleware` does on each `return input`) is the standard pattern for letting custom-scheme webviews talk to a same-process HTTP server — the renderer process loads its UI under the `oc://renderer` custom protocol scheme, the embedded HTTP server runs under `http://127.0.0.1:<port>`, and without the allow-list entry the browser's CORS preflight rejects every fetch from the renderer to the local server with the standard "Origin oc://renderer is not allowed" failure.

The fix is the right size — it matches the existing pattern in the same function at `:75-78` exactly, no abstraction-creep, no new `OriginAllowList` type, no env var, no config knob. The `startsWith` choice (rather than exact `===`) mirrors the localhost/127.0.0.1 arms above which need port-suffix tolerance, but the renderer scheme doesn't carry a port so technically `===` would be tighter; the inconsistency is visible enough to be worth a comment.

Two nits that are real but small enough not to gate merge: (1) `startsWith("oc://renderer")` will *also* match `oc://renderer.evil.com` — the `://` boundary protects against the typical sub-host attack, but a future custom-scheme that's a path-prefix of another scheme (`oc://render` matching `oc://renderer`) would silently widen the allow-list; an exact `===` for the no-port-no-path case would be safer. (2) The five allow-list arms in this function are now starting to look like a list-data structure pretending to be code — the fourth or fifth time someone adds a custom-scheme entry, lifting these to an array of `(scheme, predicate)` tuples plus a single loop will read better and let the test surface enumerate the exact set in CI.

## what I learned
Custom-scheme + same-process-HTTP is one of the most common Electron/Tauri/webview integration patterns and one of the most under-tested CORS surfaces — the dev sees it work in dev because the renderer loads under `http://localhost:5173` (Vite dev server, automatically allow-listed), then breaks in production because the renderer ships under the custom scheme. CORS allow-list additions for these are almost always one-line and almost always merit a unit test pinning the exact set of allowed schemes, because the next "let me clean up this allow-list" refactor will silently drop entries.
