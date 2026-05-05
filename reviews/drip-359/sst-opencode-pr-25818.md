# sst/opencode PR #25818 — Type session not-found errors

- Repo: `sst/opencode`
- PR: #25818
- Head SHA: `6b5dff177933ed95f714f7cf55b68cadafdbfe08`
- Author: `kitlangton`
- Updated: 2026-05-05T04:29:34Z
- Verdict: **needs-discussion**

## What it does

Adds a new design-doc spec at `packages/opencode/specs/effect/errors.md` (+329 lines) describing a phased migration from the current grab-bag of `NamedError.create(...)`, `namedSchemaError(...)`, `class extends Error`, `throw`, and `Effect.die(...)` toward typed Effect service errors via `Schema.TaggedErrorClass`, plus an explicit HTTP boundary using `HttpApiSchema.status(...)` annotations.

The PR title says "Type session not-found errors" but the diff is **only the spec doc** — no actual `Session` service typing changes, no `httpapi/errors.ts` module, no migrated route group. The first vertical-slice migration (Phase 3 in the doc) is described but not implemented.

## Specific references

- `specs/effect/errors.md:38-50` — defines the desired `SessionBusyError` shape:
  ```ts
  export class SessionBusyError extends Schema.TaggedErrorClass<SessionBusyError>()(
    "SessionBusyError",
    { sessionID: SessionID, message: Schema.String },
  ) {}
  export type Error = Storage.Error | SessionBusyError
  ```
- `specs/effect/errors.md:65-73` — pins error→status via `{ httpApiStatus: 404 }` declaration annotation rather than a name-to-status map.
- `specs/effect/errors.md:131-142` — explicitly forbids `Effect.die(...)` for "user, IO, validation, missing-resource, auth, provider, worktree, or busy-state failures" and bans `catchDefect` for expected-error recovery.
- `specs/effect/errors.md:175-185` — Phase 1 ("Stabilize the Bridge") includes a useful operational change: "Stop returning stack traces in unknown HTTP `500` responses; log the full `Cause.pretty(cause)` server-side instead." This alone is a leak-fix worth landing standalone.
- `specs/effect/errors.md:200-214` — Phase 3 names `httpapi/handlers/session.ts` as the first slice and calls out replacing `catchDefect` there with typed error mapping, plus adding endpoint error schemas — none of this code is in the diff.

## Concerns

1. **Title vs. content mismatch.** A PR titled "Type session not-found errors" that ships only a 329-line plan document is hard for a reviewer to validate against runtime behavior. Either rename to "spec(effect/errors): plan typed-error migration" or land it together with the first Session vertical slice.
2. **Spec is normative but unenforced.** Rules like "Do not use `Effect.die(...)` for … missing-resource … failures" are good policy but there's no lint rule, no codeowner gate, and no migration checklist file — without an enforcement hook these tend to drift the moment the spec author moves on.
3. **`HttpApiSchema.status(...)` vs. endpoint `error: [...]` interaction is underspecified.** The doc says "The status annotation is only used if the error is part of the endpoint, group, or middleware error schema and the handler fails with that error on the typed error channel" — that's the Effect `HttpApi` contract, but a worked example showing what happens to a typed error that is *not* declared on the endpoint (defect path? generic 500? bridge-middleware path?) would prevent the next contributor from re-introducing the same `catchDefect` pattern this spec is trying to delete.
4. **Legacy `{ name, data }` body compatibility** is mentioned (Phase 2) but the actual SDK consumers that depend on this wire shape aren't enumerated — without that list, deciding when the bridge can be removed (Phase 1's stated end-state) is ungrounded.

## Verdict
**needs-discussion** — the spec is well-reasoned and the rules section is the right policy, but the PR as submitted is a doc-only change with a title that promises code; the maintainers should decide whether to land the plan standalone (rename + merge), require it to ship with the first slice (Session not-found), or move it to a tracking issue and let the actual `Session.BusyError`/`SessionNotFoundHttpError` PR be the landing artifact.
