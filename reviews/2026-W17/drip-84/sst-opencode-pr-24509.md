---
pr: 24509
repo: sst/opencode
title: "fix: preserve enterpriseUrl through OAuth/PAT callback for self-hosted GitLab"
url: https://github.com/sst/opencode/pull/24509
head_sha: 5bfbade5ccb14536d15c48d3f64546553a2432ce
author: deve1993
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24509 — preserve `enterpriseUrl` through OAuth/PAT callback for self-hosted GitLab

Self-hosted GitLab (Duo Pro/Max) auth lands a token but the next request to the
enterprise host comes back `401 Invalid token`, because the URL of the
self-hosted instance is dropped between the auth callback and the saved
credential record. Two-bug fix: extend `Api` schema and the `oauth` arm of
`auth.ts` so `enterpriseUrl` round-trips into stored auth state.

## What the diff actually does

`packages/opencode/src/auth/index.ts:23-26` — adds
`enterpriseUrl: Schema.optional(Schema.String)` to the `Api`
`Schema.Class<Api>("ApiAuth")` definition. Optional, so existing API auth
records keep validating.

`packages/opencode/src/provider/auth.ts:200-216` — two paths:

1. The `"key" in result` branch (PAT):
   ```ts
   yield* auth.set(input.providerID, {
     type: "api",
     key: result.key,
     enterpriseUrl: result.provider,
   })
   ```
   `result.provider` here is being repurposed as the enterprise base URL.
   That's load-bearing for whatever produced `result` — needs to actually
   carry the URL string for self-hosted, not the provider literal `"gitlab"`.

2. The `"refresh" in result` branch (OAuth):
   ```ts
   const { type: _, provider, refresh, access, expires, enterpriseUrl: directEnterpriseUrl, ...extra } = result
   yield* auth.set(input.providerID, {
     type: "oauth",
     access,
     refresh,
     expires,
     enterpriseUrl: directEnterpriseUrl ?? provider,
     ...extra,
   })
   ```
   Previously `provider` was destructured into `__` and discarded. Now it's
   used as a fallback when `directEnterpriseUrl` isn't present. This recovers
   the enterprise URL for callbacks that pass it through `result.provider`
   while still letting the upstream OAuth flow override via an explicit
   `enterpriseUrl` field.

`packages/sdk/js/src/gen/types.gen.ts:1666` and `packages/sdk/openapi.json:11970`
— regenerated SDK + OpenAPI to match the schema change.

## Observations

1. **Semantics of `result.provider` are now overloaded.** In the PAT arm,
   `result.provider` is being assigned to `enterpriseUrl` unconditionally.
   For non-enterprise GitLab (or any provider that doesn't have an
   `enterpriseUrl` concept), this writes the provider *name* (`"gitlab"`,
   `"github"`, …) into a field documented as a URL. Subsequent code that
   does `new URL(auth.enterpriseUrl)` will throw. Two reasonable fixes:
   (a) only write `enterpriseUrl` when `result.provider` is a URL
   (`result.provider?.startsWith("http")`), or (b) rename the upstream
   field that produces `result.provider` so this code can distinguish
   "provider id" from "enterprise base URL". Option (a) is the cheap fix
   that keeps this diff small.

2. **No test for the api/PAT path.** The OpenAPI spec gained an
   `enterpriseUrl` property and the schema gained an optional field, but
   I don't see a test that exercises the PAT branch on a self-hosted host
   and asserts the saved record contains the URL. A 5-line test in
   `packages/opencode/test/auth/*` that calls `auth.set` and reads back
   would lock the contract.

3. **No migration / read-side test for old API records.** Existing users
   on the previous schema have `Api` records with no `enterpriseUrl`.
   Schema is `Schema.optional` so they parse, but any downstream code that
   *requires* `enterpriseUrl` for self-hosted will silently fall back to
   the stock domain. Worth one assertion that "self-hosted record
   re-loaded after restart still hits the right host" — that's what this
   PR is supposed to be fixing end-to-end and the issue body claims it
   does, but it's not in the diff.

4. **`...extra` after `enterpriseUrl` is fine but fragile.** If the
   upstream `result` ever has its own `enterpriseUrl` *and* it's destructured
   out separately as `directEnterpriseUrl`, but `extra` still contains
   another field collision, the spread at the end would clobber. Today
   that's not a problem because all the named fields are extracted, but if
   the OAuth result schema grows, this is a class of bug worth a one-line
   comment: `// directEnterpriseUrl is destructured separately so it isn't included in ...extra`.

## Verdict

**merge-after-nits.** The shape of the fix is right — `enterpriseUrl`
needed to round-trip and the Schema/SDK regeneration is consistent — but
the unconditional `enterpriseUrl: result.provider` write in the PAT arm
will pollute non-enterprise records with the provider literal. Guard it
behind a URL-shape check (or only write it when a real enterprise URL is
known) before merging. Also add the round-trip test.

## Banned-string scan

None.
