# Review: anomalyco/opencode#25998

- **PR:** [feat(desktop): implement clipboard write permission handling](https://github.com/anomalyco/opencode/pull/25998)
- **Head SHA:** `baaf676ac11e40b19062d5c3281323788f43c4c8`
- **Merged:** 2026-05-06T08:40:46Z
- **Files:** `packages/desktop/src/main/windows.ts` (+30 / -0)
- **Verdict:** `merge-as-is`

## Summary

Adds an Electron permission handler so the renderer can write to the clipboard via the `clipboard-sanitized-write` permission, but only from a trusted origin (the custom `oc://renderer` protocol or the configured `ELECTRON_RENDERER_URL` dev server). Wired into both the main and loading windows.

## Specific notes

- `windows.ts:212-216` — request handler correctly ANDs three conditions: permission match, trusted URL, and matching webContents id. This is the right shape for a least-privilege permission gate.
- `windows.ts:217-221` — check handler short-circuits non-clipboard permissions to `false`, which means this handler is permissive only for the one permission and cannot accidentally grant anything else. Good defensive default.
- `windows.ts:225-232` — `isTrustedRendererUrl()` uses `URL.canParse` before constructing the URL — avoids throws on weird inputs. Only allows the renderer protocol+host or an exact origin match against the dev URL. `process.env.ELECTRON_RENDERER_URL` is read fresh each call, which is fine since it's set at startup.
- The `webContents.id === win.webContents.id` check on the request path means an embedded webview/iframe with a different webContents would be denied even if the URL looked trusted. That's a nice belt-and-braces guard.

## Rationale

Tightly scoped, correct, easy to reason about. Permission is granted only to: (a) the right permission name, (b) the right window, (c) the right origin. The dev URL fallback is keyed on an env var that only the dev harness sets, so prod builds reduce to "renderer protocol only." Test coverage would be nice but the surface is tiny and the diff is self-evidently correct.
