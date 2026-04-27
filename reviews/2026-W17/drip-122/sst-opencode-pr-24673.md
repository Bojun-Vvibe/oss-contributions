---
pr: 24673
repo: sst/opencode
sha: 7d48e3549d103dd10c856cf8c14b976d5ca5122b
verdict: merge-after-nits
date: 2026-04-28
---

# sst/opencode #24673 — fix(app): handle missing session route

- **Author**: kill74 (Guiii)
- **Head SHA**: `7d48e3549d103dd10c856cf8c14b976d5ca5122b`
- **Size**: 79 added / 7 deleted across 1 file (`packages/app/src/pages/session.tsx`).
- **Closes**: #23514

## Scope

When the desktop/web app keeps a stale `/session/:id` route after switching
projects or reconnecting to a different opencode server, the in-flight
`sync.session.sync(id)` resource throws a `NotFoundError` that propagates
to the SolidJS error boundary and surfaces as a crash screen. Fix detects
the missing-session error specifically, shows a toast, and navigates back
to the parent route. Also silences the same error class in the scheduled
refresh path so a backgrounded stale tab stops emitting console noise.

## Specific findings

- `packages/app/src/pages/session.tsx:88-130` — the new
  `isSessionNotFoundError(error, sessionID)` helper walks `error.cause`
  to handle the SDK's wrapping pattern (where the original `NotFoundError`
  is buried under one or more re-throws). The walk handles three error
  shapes: string, `Error` instance, and plain `{name, message, cause}`
  object. Correct shape — SDK errors in this codebase have come back in
  all three forms historically.
- `packages/app/src/pages/session.tsx:97-99, 109-111, 122-124` — the
  match predicate is repeated three times across the branches:
  `name === "NotFoundError" || message.includes("Session not found") ||
  (message.includes(sessionID) && message.toLowerCase().includes("not found"))`.
  This is a defensible OR-of-three (covers SDK-classed, server-message,
  and message-contains-id-and-not-found shapes) but extracting one
  `matchesNotFound(name, message, id)` predicate would dedup the three
  branches and make the contract auditable in one place.
- `packages/app/src/pages/session.tsx:91-95` — the docstring honestly
  names the SDK-wraps-cause pattern as the reason for the walk. Good.
- `packages/app/src/pages/session.tsx:817-825` — scheduled refresh path:
  `void sync.session.sync(id, { force: true }).catch((error) => { if
  (isSessionNotFoundError(error, id)) return; console.debug(...) })`.
  Correct: silence the missing-session class but still log other
  refresh failures at `debug` level so they're not lost. The
  `console.debug` (rather than `console.error`) is the right choice
  for a backgrounded refresh — the foreground catch already surfaces
  user-visible errors.
- `packages/app/src/pages/session.tsx:830-849` — foreground try/catch:
  on `isSessionNotFoundError`, only act if the route is still pointing
  at the same `id` (`if (params.id !== id) return ""`) — guards against
  the user navigating away mid-fetch and getting a stale toast. Then
  `showToast({variant: "error", title: "Session not found", description:
  "This session does not exist on the connected server."})` and
  `navigate("../", { replace: true })`. The `replace: true` is
  load-bearing: without it, the user's "back" button would re-navigate
  to the missing session.
- `packages/app/src/pages/session.tsx:803-804, 833, 850` — return type is
  `string | undefined` and the success branch now returns `""` (empty
  string) instead of the actual sync result. Reading the surrounding
  code, `sessionSync()` is consumed only as a `createResource` loading
  signal (`sessionSync.loading`, etc.), not for its value. So returning
  `""` is fine — but it should be commented, because a future reader
  will absolutely wonder why we throw away the result. A `// returned
  value is unused; consumers only read .loading/.error` comment at the
  top of the resource fetcher would close that gap.

## Risk

Low-medium. The fix is correctly scoped to one specific error class with
guards against navigate-after-route-change, and the error walk is
defensive against the three known SDK error shapes. Two real nits:
(1) the triple-repeated match predicate is a maintenance hazard — when a
fourth error shape appears, three sites need updating; (2) `return ""`
versus the prior `return sync.session.sync(id)` discards the resource
value silently. Neither blocks merge but both are cheap to fix.

The `navigate("../", { replace: true })` assumes the route hierarchy
puts a sensible parent above `/session/:id`. If a future routing
refactor moves session routes under a different parent, this becomes
silent navigate-to-wrong-place. Worth a comment naming the route
contract or a routing constant.

## Verdict

**merge-after-nits** — extract the triple-repeated `(name, message)`
predicate into one `matchesNotFound` helper; add a comment on the
`return ""` discard explaining it's intentional; consider naming the
parent-route assumption. The core fix shape is right.

## What I learned

The `error.cause` walk + multi-shape match for SDK-wrapped errors is a
pattern worth standardizing. Every SolidJS resource that calls into an
SDK that may throw a known classified error needs roughly this same
50-line helper to (1) walk `cause`, (2) tolerate string/Error/plain-object
shapes, and (3) match by class-name plus message-substring. A shared
`util/sdkError.ts` with a `matches(error, classifier)` API would let
the next route-error PR be 5 lines instead of 50.
