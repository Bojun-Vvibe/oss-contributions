---
pr: 24322
repo: sst/opencode
sha: 29eeeda12847d2859515014af480cf228350e8cd
verdict: merge-as-is
date: 2026-04-25
---

# sst/opencode#24322 — fix(permission): reject stale permission replies

- **URL**: https://github.com/sst/opencode/pull/24322
- **Author**: pascalandr
- **Closes**: #15386

## Summary

`Permission.reply()` previously returned silently when a `requestID`
wasn't in the in-memory `pending` map (e.g. after a process restart),
so HTTP clients got `200 / true` for permission IDs that no waiter had
ever heard of. This PR flips that path to throw `NotFoundError` and
maps it through the experimental `HttpApi` route to a proper 404.

## Reviewable points

- `packages/opencode/src/permission/index.ts:218` — the load-bearing
  change: the `if (!existing) return` branch becomes
  `if (!existing) throw new NotFoundError({ message: ... })`. Note
  this uses `throw` inside an `Effect.fn` — Effect catches sync
  throws and turns them into the typed error channel, but the
  signature update on line 130 (`reply` now returns
  `Effect.Effect<void, InstanceType<typeof NotFoundError>>`) is what
  actually gives downstream callers a typed failure they can pattern
  on. Both halves are needed; PR has both. Good.

- `packages/opencode/src/server/routes/instance/httpapi/permission.ts:27`
  adds `error: HttpApiError.NotFound` to the endpoint schema, and the
  handler at line ~64 now does
  `.pipe(Effect.catchIf(NotFoundError.isInstance, () => Effect.fail(new HttpApiError.NotFound({}))))`.
  That's exactly the right boundary translation: storage-layer
  `NotFoundError` should never leak as-is to the HTTP layer; mapping
  to the framework's `HttpApiError.NotFound` makes the OpenAPI-spec
  surface honest.

- `packages/opencode/test/permission/next.test.ts:1049` renames
  `"reply - does nothing for unknown requestID"` →
  `"reply - fails for unknown requestID"` and asserts
  `expect(err).toBeInstanceOf(NotFoundError)`. The rename matters
  because the old test name encoded the bug as the contract — anyone
  doing a future "this test is failing, what was it asserting?"
  archaeology now sees the correct contract.

## Rationale

Tiny, surgical bug fix with the right test rewrite, the right
boundary translation at the HTTP layer, and a real issue close
(#15386). No reason to hold this. The `NotFoundError` re-export from
`@/storage` is already public (used elsewhere in the codebase), so
re-using it for the permission domain is fine even though the request
isn't strictly a "storage" miss — it's the standard not-found shape
this codebase already uses for HTTP 404 mapping.

## What I learned

A handler that returns `void` on a "request not in our map" lookup is
almost always a latent bug — silent success is the worst possible
answer to "did my reply land?". The fix here is a useful template:
(1) flip the silent-return to a typed error in the service layer,
(2) widen the Effect signature so callers must acknowledge the new
failure mode at the type level, (3) at the HTTP boundary,
`catchIf(<DomainError>.isInstance, ...)` and translate to the
framework's standard 404. That third step is the one most often
skipped — without it the typed error escapes as a generic 500 and
you've just moved the silent-bug problem one layer out.
