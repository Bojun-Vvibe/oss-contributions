# sst/opencode PR #25358 — Preserve workspace adapter context

- URL: https://github.com/sst/opencode/pull/25358
- Head SHA: `38a44903c52dafc31bdca28505a0607a3af59290`
- Author: kitlangton
- Verdict: **merge-after-nits**

## Summary

Refactors `packages/opencode/src/control-plane/adapters/{index.ts,worktree.ts}` to split workspace adapters into two layers: an external `WorkspaceAdapter` plugin contract (Promise-based, exposed via `@opencode-ai/plugin`) and an internal `InternalWorkspaceAdapter` (Effect-based, threading the `WorktreeService` through Effect's dependency-injection layer instead of the previous lazy `import("@/effect/app-runtime")` + `AppRuntime.runPromise` pattern). The net effect is that workspace adapter code now runs inside the caller's Effect context rather than escaping to a fresh runtime, which fixes context loss for span/log/tracer scopes.

## Line-level observations

- `adapters/index.ts` lines 8–13: `BUILTIN` becomes a `WorkspaceAdapterEntry[]` (just `{type, name, description}`) and a separate `makeBuiltinAdapters(worktree)` factory returns `Map<string, InternalWorkspaceAdapter>`. The split correctly separates *metadata* (which `listAdapters` exposes without needing a service) from *behavior* (which requires a `WorktreeService`).
- `getAdapter` now takes `builtin: ReadonlyMap<string, InternalWorkspaceAdapter> = emptyBuiltinAdapters`. The default of `emptyBuiltinAdapters` means callers who forget to pass the built-in map will get `Unknown workspace adapter: worktree` for the worktree case. Consider failing louder, or making the parameter required.
- `fromPromiseAdapter` (lines ~36–60) wraps each plugin method with `runPromiseAdapter` → `EffectBridge.make() ... bridge.run(Effect.tryPromise(...))`. The `Promise.resolve().then(fn)` pattern correctly normalises sync-throwing plugins, but it loses sync stack traces; if a plugin throws synchronously, the rejection chain will report the location of `Promise.resolve().then` rather than the plugin. Consider `try { return await fn() } catch (e) { ... }` inside `tryPromise`'s `try`.
- `adapters/worktree.ts` lines 32–40: `catchWorktreeError` correctly rethrows interrupt-only causes via `Effect.failCause` and converts everything else to `WorkspaceAdapterError`. Good defensive split between Effect's interruption signal and real failures.
- `decodeConfig` uses `Effect.try` with `cause instanceof Error ? cause.message : String(cause)` — fine, but `WorkspaceAdapterError` already accepts `cause`, so the message-only lossy stringification is a small regression in observability.

## Suggestions

1. Make the `builtin` parameter on `getAdapter` non-optional — the default-empty-map silently hides bugs.
2. In `fromPromiseAdapter`, prefer `try: async () => fn()` so synchronous throws in plugin code surface with their original stack.
3. In `decodeConfig` / `adapterError`, pass `cause` straight through rather than stringifying — it propagates better for Effect tracing.
