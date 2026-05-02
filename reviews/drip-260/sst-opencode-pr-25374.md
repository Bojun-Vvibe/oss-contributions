# sst/opencode PR #25374 — Drop ALS fallbacks from containsPath and workspace routing

- PR: https://github.com/sst/opencode/pull/25374
- Head SHA: `f9a3d0baee77c953f763694caa58876aeef6a2d2`
- Author: @kitlangton
- Size: +20 / -31
- Status: OPEN

## Summary

Two unrelated dead-code removals from the AsyncLocalStorage reader audit:
1. `Instance.containsPath(filepath, ctx?)` made `ctx` required because every caller already passed one — the implicit `Instance.directory` / `Instance.worktree` ALS reads were unreachable.
2. `WorkspaceRoutingMiddleware.currentDirectory()` had a try/catch around `Instance.directory` that ran during the routing phase before any instance is bound. The try always threw; the catch always returned `process.cwd()`. Replaced by direct `process.cwd()`.

## Specific references

- `packages/opencode/src/project/instance.ts:33-41` — signature changes from `containsPath(filepath, ctx?)` to `containsPath(filepath, ctx)`, body drops the `ctx ?? Instance` fallback. Compile-time enforcement that callers pass context.
- `packages/opencode/src/config/config.ts:461` — call site simplifies from `InstanceRef.use((ctx) => Effect.succeed(Instance.containsPath(source, ctx)))` to direct `Instance.containsPath(source, ctx)` (a `ctx` is already in scope from the surrounding `Effect.gen`). Cleaner and avoids the unnecessary Effect wrap.
- `packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:45-52` — `currentDirectory()` helper deleted; `defaultDirectory()` now ends with `|| process.cwd()` directly.
- `packages/opencode/test/file/path-traversal.test.ts:131-198` — every test in `describe("Instance.containsPath", …)` updated to pass `Instance.current` explicitly. 7 call sites, all mechanical.

## Verdict: `merge-after-nits`

Diagnosis is sound — the try/catch path was provably dead given that `WorkspaceRoutingMiddleware` runs ahead of `InstanceRef` provision — and the call-site audit looks complete. The PR description even calls out the remaining design question (server-bound root via `Context.Reference`) as a separate change, which is the right scope discipline.

## Nits / concerns

1. **`Instance.current` semantics in tests.** The new tests pass `Instance.current` inside `Instance.provide({ directory, fn })`. Worth confirming `Instance.current` is the same object the production caller in `config.ts:461` would receive (`ctx` from the `Effect.gen` scope) — i.e. not a new snapshot vs. live ref divergence. A one-line comment in `instance.ts` explaining what `Instance.current` returns relative to `InstanceContext` would help future readers.
2. **`process.cwd()` as the routing fallback is subtle.** The PR description correctly notes this matches the legacy Hono `InstanceMiddleware` (`server/routes/instance/middleware.ts:11`). But the legacy file path should be linked as a code reference in a comment in `workspace-routing.ts:58-59`, otherwise the next reader who wants to "fix" it by reading the experimental server-bound directory won't know about the parity constraint.
3. **No migration note for external callers of `containsPath`.** This is a public-ish helper on `Instance`. If anyone outside `packages/opencode` imports it (plugins?), the signature break is a surprise. A search across the monorepo plus a one-line CHANGELOG note would close that out.
