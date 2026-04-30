# sst/opencode #25074 — feat(httpapi): add CORS middleware to instance routes

- **Author:** Brendonovich (Brendan Allan)
- **SHA:** `71837cf`
- **State:** MERGED
- **Size:** +18 / -0 across 1 file (`packages/opencode/src/server/routes/instance/httpapi/server.ts`)
- **Verdict:** `merge-after-nits`

## Summary

Adds `HttpRouter.cors({ maxAge: 86_400, allowedOrigins: <predicate> })` provided as a
layer to the merged `routes` at `packages/opencode/src/server/routes/instance/httpapi/server.ts:107-122`.
The `allowedOrigins` predicate accepts: empty/no origin (curl-style same-origin),
`http://localhost:*`, `http://127.0.0.1:*`, `oc://renderer`, the three Tauri origin
shapes (`tauri://localhost`, `http://tauri.localhost`, `https://tauri.localhost`), and
any subdomain matching `^https://([a-z0-9-]+\.)*opencode\.ai$`.

## Reasoning

The allowlist is the right shape for a desktop+browser+native renderer story — each
origin in the union maps to a known production surface. The 24h `maxAge: 86_400` is
the typical browser cap and avoids preflight chatter. The predicate is correctly
narrow: it does NOT do a `startsWith("http://")` blanket match, so the
`localhost:`/`127.0.0.1:` discipline still requires the trailing colon. Two nits worth
addressing before this becomes the established pattern: (1) the `as Predicate<string>
as any` double-cast at `:120` papers over an `effect/Predicate` typing gap that should
be filed upstream rather than silently cast-erased — a one-line `// TODO: drop the
as-any once @effect/platform exports HttpRouter.cors with the right Predicate signature`
comment would make the debt visible; (2) the regex `^https:\/\/([a-z0-9-]+\.)*opencode\.ai$`
permits zero subdomain segments which means bare `https://opencode.ai` is allowed
alongside `https://www.opencode.ai` / `https://share.opencode.ai` — that's likely
intentional but worth a one-line comment so the next reviewer doesn't tighten it to
require a leading subdomain. No regression test for the predicate is included; the
preflight surface is large enough (8 origin shapes) that a `describe.each` would be
cheap insurance. None of these are blocking.
