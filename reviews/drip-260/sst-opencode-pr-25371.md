# sst/opencode PR #25371 — fix: accept workspace create payload without extra

- PR: https://github.com/sst/opencode/pull/25371
- Head SHA: `b7058f3cefe4e17dfbe644b146a62dc73aa91d46`
- Author: @kitlangton
- Base: `kit/instance-loader-service`
- Size: +56 / -3
- Status: MERGED

## Summary

The new HttpApi workspace create endpoint required `extra` in the request body, but the TUI client (and the legacy Hono route still in production) sends `{type, branch}` without it. The `Schema.Struct(Struct.omit(Workspace.CreateInput.fields, ["projectID"]))` derivation made `extra` mandatory by reflection. Fix marks `extra` optional in the schema and normalizes missing/undefined to `null` in the handler before calling `workspace.create`. Adds a red-then-green test plus a parallel test asserting the legacy Hono path accepts the same shape.

## Specific references

- `packages/opencode/src/server/routes/instance/httpapi/groups/workspace.ts:11-15` — schema rebuild: `Struct.omit(..., ["projectID", "extra"])` then re-add `extra: Schema.optional(Workspace.CreateInput.fields.extra)`. Correct primitive.
- `packages/opencode/src/server/routes/instance/httpapi/handlers/workspace.ts:27` — `extra: ctx.payload.extra ?? null` normalizes undefined → null so downstream `Workspace.create` (which expects `extra: ...|null`) doesn't see undefined.
- `packages/opencode/test/server/httpapi-workspace.test.ts:198-219` — new HttpApi-flagged test asserting `{type, branch:null}` returns 200 with `extra: null`.
- `packages/opencode/test/server/httpapi-workspace.test.ts:221-247` — parallel test with `httpApi=false` asserting legacy Hono parity. Good belt-and-suspenders against future regression.
- `packages/opencode/test/server/httpapi-workspace.test.ts:30-32` — `request()` gains an `httpApi` param toggling `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI`. Reasonable plumbing.

## Verdict: `merge-as-is`

Tight 3-file change with red-then-green coverage on both the new and legacy paths. Both reviewer concerns I'd normally raise (schema vs handler symmetry; legacy parity) are already addressed in the diff.

## Notes (non-blocking, post-merge)

1. The `request()` helper now mutates `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` as a side effect on every call. That's pre-existing, but the new `httpApi=false` overload makes the implicit ordering across tests (whichever ran last wins for any subsequent non-`request()` call) more visible. Worth a `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` reset in an `afterEach` at some point.
2. Consider also asserting that `extra: undefined` (explicit) and `extra` missing decode identically — the optional-schema decode behavior is the kind of thing Effect Schema sometimes surprises on.
