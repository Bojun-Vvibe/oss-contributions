# sst/opencode PR #25099 — fix(opencode): allow oc://renderer origin in cors middleware

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/25099
- Head SHA: `47d46897d97d2ccead980e332686e1d4578d6eab`
- State: MERGED (2026-04-30T12:11:43Z)
- Files: 1 (`packages/opencode/src/server/middleware.ts`), +1/−0
- Verdict: **merge-as-is** (already merged; this is a post-merge note)

## What it does

Adds `oc://renderer` to the `CorsMiddleware` allow list at
`packages/opencode/src/server/middleware.ts:77`, sandwiched between the
existing `localhost:` / `127.0.0.1:` allowances and the `tauri://`
renderer-protocol allowances. Pure additive change — one line, no
behavior change for any other origin.

## Notes

- The new `oc://renderer` allowance is structurally identical to the
  existing `tauri://localhost` allowance: both are non-`http(s)`
  custom-protocol URLs used by an embedded webview to talk back to the
  same-process HTTP server. The placement (right next to the `tauri://`
  arms) telegraphs the intent — this is the desktop renderer custom
  scheme analogue of tauri.
- `startsWith("oc://renderer")` (no trailing `/`) is intentionally loose
  enough to match `oc://renderer`, `oc://renderer/`, and
  `oc://renderer/foo/bar` as the same origin family. That matches how
  `http://localhost:` is matched (port-prefix only, no path discipline).
- No new dependency, no new state, no new test surface — but also no
  test coverage of the new arm.

## Nits (post-merge follow-ups)

1. Missing test. The CORS middleware suite (if any) should pin this
   arm so the next "tidy the allow list" refactor doesn't drop it. A
   one-liner asserting `cors("oc://renderer/index.html") === "oc://renderer/index.html"`
   would be enough.
2. Consider tightening to `oc://renderer/` (with trailing slash) so that
   `oc://renderer.evil.example` cannot match. Custom-scheme attacks are
   uncommon but the prefix match is loose; a `/` suffix or an exact
   `=== "oc://renderer"` plus `startsWith("oc://renderer/")` pair would
   close that. Symmetric with the `tauri://localhost` exact-match arm
   right below.
3. No comment naming who emits `oc://renderer` (presumably the desktop
   shell wrapping the webview). A one-line `// desktop renderer custom
   scheme — see <link to renderer registration>` would orient the next
   reader.

## Why merge-as-is

Already merged. One-line additive allowance whose intent is obvious
from neighboring lines, and the prior PR (#25074, drip-211) added the
CORS middleware to instance routes in the first place — this is just
filling in the renderer-protocol arm that #25074 didn't yet need.
