# Review: sst/opencode #25636 — fix(app): preserve auth token credentials

- PR: https://github.com/sst/opencode/pull/25636
- Head SHA: `a01cfa124a9468195a94162458f0d26064e53b1f`
- Author: kitlangton
- Files touched: `packages/app/src/components/terminal.tsx`, `packages/app/src/context/server.test.ts` (new), `packages/app/src/context/server.tsx`, `packages/app/src/entry.tsx`, `packages/app/src/utils/server.test.ts` (new)

## Verdict: `merge-after-nits`

## Rationale

Solid bug fix. The core issue: when the app boots with `?auth_token=...` it constructs a `ServerConnection.Http` with credentials but the existing `allServers` memo at `server.tsx:113` always concatenated `props.servers` *after* `store.list`, so the persisted bare-URL entry shadowed the prop entry in the dedup `Map` (last write wins, but iteration order put stored first). The fix extracts `resolveServerList` (`server.tsx:36-60`) and reverses the precedence so `props` comes first — verified by the new test at `context/server.test.ts:5-30` which asserts the credentialed `props` entry survives. The `clearAuthToken` helper in `entry.tsx:115-121` correctly scrubs the query param via `history.replaceState` before render so creds don't sit in the URL bar. Two nits: (1) `clearAuthToken` builds the URL with `(params.size ? \`?${params}\` : "")` — `URLSearchParams.toString()` is fine but explicit `params.toString()` reads better and is less ambiguous to readers expecting object coercion semantics. (2) The new `authToken?: boolean` field on `Http` (`server.tsx:69`) is just a marker but its semantics aren't documented — a one-line JSDoc explaining "set to true when credentials originated from a one-time `?auth_token` param so callers know not to persist them" would prevent the next reader from treating it as the token itself. Test coverage is appropriate; merge after the JSDoc.