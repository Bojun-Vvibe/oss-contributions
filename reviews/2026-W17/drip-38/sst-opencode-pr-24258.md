# sst/opencode #24258 — feat(httpapi): bridge instance read endpoints

- **Repo**: sst/opencode
- **PR**: [#24258](https://github.com/sst/opencode/pull/24258)
- **Head SHA**: `31af9dcab87dff8f8010d9eefc5abf737d831349`
- **Author**: kitlangton
- **State**: OPEN (+194 / -37)
- **Verdict**: `merge-as-is`

## Context

Continues the in-flight migration from Hono routes to Effect's
experimental `HttpApi` (tracker spec at `packages/opencode/specs/
effect/http-api.md`). This slice bridges three top-level instance
read endpoints — `/path`, `/vcs`, `/vcs/diff` — through the
`HttpApi` server while keeping the existing Hono surface working
unchanged via dual schema registration.

## Design

The interesting trick is the **schema-with-statics** pattern that
keeps both Hono+Zod and Effect+Schema callers happy off a single
source-of-truth definition.

1. `Vcs.Mode`, `Vcs.Info`, `Vcs.FileDiff` (`packages/opencode/src/
   project/vcs.ts:104-134`) move from `z.enum` / `z.object` to
   `Schema.Literals` / `Schema.Struct`. Each is then `.pipe`d
   through `withStatics((s) => ({ zod: zod(s) }))` — so
   `Vcs.Info.zod` continues to work where the old Hono validator
   stack expects a Zod schema. `instance/index.ts:148, 174, 184`
   show the call sites: `resolver(Vcs.Info.zod)`, `resolver(Vcs.
   FileDiff.zod.array())`, `mode: Vcs.Mode.zod`.

2. `InstanceApi` (`packages/opencode/src/server/routes/instance/
   httpapi/instance.ts:33-71`) defines the three endpoints as a
   single `HttpApiGroup` with explicit `Authorization` middleware
   and OpenAPI annotations on each endpoint. Path constants
   exported from `InstancePaths` (lines 27-31) so the Hono
   shim can re-bridge them in `instance/index.ts:57-59`.

3. `instanceHandlers` (lines 73-103) is a single `Layer.unwrap`
   that yields the `Vcs.Service` once and wires three handler
   fns. `getVcs` (lines 87-90) does the right thing — runs
   `vcs.branch()` and `vcs.defaultBranch()` with `concurrency: 2`
   instead of awaiting them serially. Small but meaningful: those
   are independent git invocations.

4. The Hono side at `instance/index.ts:54-60` mounts the bridge
   under the `Flag.HTTPAPI_BRIDGE` carve-out (existing pattern
   from earlier slices). Same handler is reachable via Hono path
   matching, just dispatched through the Effect server.

5. New test file `test/server/httpapi-instance.test.ts:1-53` adds
   bridge coverage for path/VCS reads — referenced in the PR body
   alongside the run command.

## Risks

- Dual schema registration (`zod` static + Effect Schema) means
  schema drift is now a real risk: change one, forget the other.
  Mitigated by the `withStatics` pattern keeping them in the same
  file and forcing both to be touched in one diff. Still worth a
  follow-up that derives the Zod side from the Effect side
  programmatically — `zod(s)` already does this; it's the type
  assertions in the Hono handlers that lock the duplication.
- `getVcsDiff` (line 92-94) types `ctx.query.mode` as `Vcs.Mode`;
  Effect's query parser does the schema validation. The Hono
  side at line 184 still goes through `validator("query", z.object
  ({ mode: Vcs.Mode.zod }))`. Both correct, but if a request hits
  the Effect bridge with a malformed `mode`, the error shape may
  differ from the Hono side. Cross-surface error parity is on
  the spec's "Next PRs" list anyway.
- Spec markdown updates (`http-api.md:142-167`) flip
  `top-level instance reads` from `next` to `bridged partial`,
  and `workspace` from `implemented` to `bridged`. Make sure the
  workspace change is actually exercised by an existing test —
  the diff also flips the spec checkbox (`Add bridge-level auth
  and instance tests` → `[x]`) which suggests yes.

## Suggestions

None blocking. Nit: the OpenAPI annotation at lines 38-42 spells
"OpenCode" with the canonical capitalization; rest of the codebase
uses `opencode` lowercase consistently. Minor.

## What I learned

The `withStatics` pattern is a nice middle ground for in-flight
migrations between two schema systems: instead of maintaining
parallel definitions or doing a flag-day flip, you attach the
"old shape" as a static getter on the new schema and let call
sites migrate at their own pace. The cost is that grep for
`z.object` keeps finding hits in the route layer until the
migration is done, but it's a correct cost — the Zod surface is
still the wire contract for unbridged routes.
