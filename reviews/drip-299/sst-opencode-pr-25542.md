# sst/opencode PR #25542 — refactor(server): extract Hono-coupled utilities to backend-neutral modules

- URL: https://github.com/anomalyco/opencode/pull/25542
- Head SHA: `2ea74043660c5d2657ea892161c2fbff1b9206fd`
- Author: kitlangton (Kit Langton)

## Summary

Pulls the implementation of `fence`, `tui-control`, `workspace-routing`, and
`ui` out of their Hono-flavored homes under `packages/opencode/src/server/`
into a `server/shared/` layer, leaving the original modules as thin
re-export shims. Part of a series migrating the server toward HttpApi while
keeping Hono callers green.

## Review

This is a mechanical extract-to-shared refactor. Confirmed against the diff:

- `server/fence.ts` (rewrite at the diff head) collapses to a re-export of
  `./shared/fence`, which in turn owns the `Database`/`EventSequenceTable`/
  `Workspace`/`AppRuntime` wiring previously inline.
- `server/proxy.ts:4` switches its `Fence` import to `./shared/fence`.
- `server/routes/instance/httpapi/handlers/tui.ts:8` and `middleware/
  workspace-routing.ts` move to `@/server/shared/tui-control` and
  `@/server/shared/workspace-routing`.
- `server/routes/instance/httpapi/server.ts:48` now imports `serveUIEffect`
  from `@/server/shared/ui`.

No behavior change is intended. Two things to verify before merge:

1. The shim re-exports must include every public symbol the old module
   exposed. The `fence.ts` shim re-exports `HEADER, diff, load, parse, wait,
   waitEffect` and the `State` type — confirm none of the existing callers
   reach for anything else (e.g. an internal helper that wasn't exported but
   was reached via `Fence.*` somewhere). A repo-wide grep for
   `from "@/server/fence"` and `from ".*/server/fence"` would close this.
2. The local `log = Log.create({ service: "fence" })` is now declared in
   both the shim and the shared module (the shim still creates it but
   doesn't appear to use it). Drop the unused one in the shim to keep
   logger names from being registered twice on hot reload.

## Verdict

`merge-after-nits` — refactor is sound; verify the shim covers all
historical exports and remove the duplicated `log` factory.
