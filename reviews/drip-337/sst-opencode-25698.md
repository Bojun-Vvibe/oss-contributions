# sst/opencode #25698 — fix(opencode): strip transfer-encoding in UI proxy and allow public manifest assets

- **Head SHA reviewed:** `bd91afb3dfed9e2d53f206786820c35e41c60877`
- **Size:** +18 / -1 across 5 files
- **Verdict:** merge-after-nits

## Summary

Three small but well-scoped fixes for the embedded UI proxy:

1. `packages/opencode/src/server/shared/ui.ts:36` strips
   `transfer-encoding` from proxied response headers, fixing
   `net::ERR_INVALID_CHUNKED_ENCODING` when the upstream sends
   chunked encoding that's already been re-framed.
2. New file `packages/opencode/src/server/shared/public-ui.ts:1-12`
   defines a tiny allowlist of static UI asset paths
   (`/site.webmanifest`, two PWA icons) that bypass auth.
3. `packages/opencode/src/server/middleware.ts:48` and
   `.../instance/httpapi/middleware/authorization.ts:96` consume
   that allowlist so PWA manifest fetches don't 401 when
   `OPENCODE_SERVER_PASSWORD` is set.
4. `packages/app/src/components/terminal.tsx:484` adds `directory`
   to the PTY connect-token request so terminal startup works under
   `auth_token` flows.

## What I checked

- `public-ui.ts:6-10` — the path set is hardcoded with three entries.
  This is fine for now, but the comment should call out that *adding*
  to this set widens the unauthenticated surface area; future maintainers
  shouldn't append liberally. A short test that asserts the set is a
  *subset* of "well-known static assets" (or at least a comment listing
  the threat model) would help.
- `isPublicUIPath(method, path)` correctly restricts to `GET`. Good —
  prevents `OPTIONS` / `POST` of the same paths from sneaking through.
  Worth adding `HEAD` for completeness (browsers do issue HEAD on
  manifests in some flows).
- `middleware.ts:48` — the public-UI bypass is placed *before* the
  PTY-ticket bypass, which is the right precedence (cheaper check
  first, and more restrictive scope). No issue.
- `authorization.ts:96` — symmetric change on the HttpApi/Effect path.
  Both auth surfaces stay in lock-step, which is what the PR title
  promises ("HttpApi/Hono parity" was a separate PR but this echoes
  that pattern). Good.
- `ui.ts:36` — `transfer-encoding` deletion is correct; the existing
  comment ("transfer metadata makes browsers decode already-decoded
  assets again") covers the intent, though it would be clearer to
  split the comment so the new line reads "and chunked encoding makes
  browsers reject the response" or similar.
- `terminal.tsx:484` — adds `directory` to the connect-token payload.
  Need to confirm the server side (`pty.connectToken` handler) actually
  reads `directory` from the body; otherwise this is a silent no-op.

## Risk

Low. Allowlisting three known-static paths is a minimal expansion of
the unauthenticated surface, and the transfer-encoding strip is the
documented HTTP/1.1 behavior for re-encoding proxies.

## Recommendation

Merge after:

1. Add `HEAD` to `isPublicUIPath` (browsers HEAD manifests).
2. Add a comment / lint guard at the top of `PUBLIC_UI_PATHS`
   discouraging future additions without security review.
3. Confirm the server-side `pty.connectToken` handler binds the new
   `directory` field, ideally with a small test exercising the new
   token-scoping path.
