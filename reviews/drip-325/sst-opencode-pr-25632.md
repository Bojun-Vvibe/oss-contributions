# sst/opencode PR #25632 — fix(server): serve embedded UI from bunfs

- **PR:** https://github.com/sst/opencode/pull/25632
- **Author:** kitlangton
- **Head SHA:** `6482515f` (full: `6482515f73e421ed4986b0f34dd41b0e9de35bb8`)
- **State:** MERGED
- **Files touched:**
  - `packages/opencode/src/server/shared/ui.ts` (+26 / -13)
  - `packages/opencode/test/server/httpapi-ui.test.ts` (+34 / -1)

## Verdict

**merge-as-is**

## Specific refs

- `packages/opencode/src/server/shared/ui.ts:39-51` — new `serveEmbeddedUIEffect` extracts the embedded-asset path. The crucial change is dropping the `fs.existsSafe(match)` gate; under Bun's compiled binary the `$bunfs` virtual FS satisfies `readFile` but reports the path as missing for `access()`-style checks, which previously caused the embedded UI to 404 in production binaries. Mapping only the `PlatformError`/`NotFound` reason (`Effect.catchReason("PlatformError", "NotFound", ...)`) to a 404 while letting other errors propagate is the right surface area.
- `packages/opencode/test/server/httpapi-ui.test.ts:94-125` — regression test stubs `existsSafe` with `Effect.die("embedded UI should not rely on filesystem access checks")`, which is the exact assertion that locks in the fix. Asserting `readPath === "/$bunfs/root/assets/app.js"` and the `text/javascript` MIME also covers the call-through to `embeddedUIResponse`.

## Rationale

Tight, well-scoped fix for a real Bun-binary regression. The refactor pulls the embedded path out of `serveUIEffect` into a pure helper that takes the asset map as input — that's why the test can drive it with a synthetic `embeddedWebUI` map and a stubbed FS without standing up the full server. Error handling is narrower than the old code (only "not found" is swallowed; transient I/O errors will now surface instead of silently 404'ing), which is a strict improvement. CSP header is preserved for HTML responses (`ui.ts:35`). Nothing here looks risky; the test covers the exact regression path.

