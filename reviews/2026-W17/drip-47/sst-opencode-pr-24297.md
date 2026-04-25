# sst/opencode #24297 — fix(opencode): resolve heap unlimited + orphan processes on Linux

- **PR:** https://github.com/sst/opencode/pull/24297
- **Head SHA:** `e7673abeb1d9a93d35a9fc9248ed36b8f92b070a`
- **Files changed:** 4 — `cli/cmd/tui/context/exit.tsx` (+1/−0), `cli/cmd/tui/thread.ts` (+2/−0), `project/instance.ts` (+4/−1), `session/prompt.ts` (+4/−1)

## Summary

Closes #15348. Three independent Linux problems bundled into one tiny PR: (a) when the controlling terminal closes, the TUI process receives `SIGTERM` (not `SIGHUP` on every distro/multiplexer combo) and was never trapped, leaving an orphan; (b) the per-prompt `AbortController` was only aborted on `Effect.onInterrupt`, so on the normal success path it lived forever and pinned every fetch-related allocation, growing heap to 37 GB on long sessions; (c) the per-instance context cache was an unbounded `Map`, accumulating one entry per project visited.

## Line-level call-outs

- `packages/opencode/src/cli/cmd/tui/context/exit.tsx:58` — `process.on("SIGTERM", () => exit())` added next to the existing `SIGHUP` listener. Correct fix on the surface, but **never `off()`'d**. The `exit()` returned by `useExit` is supposed to be the public teardown hook — if a caller invokes it twice, the SIGTERM listener stays bound after the first teardown and any subsequent SIGTERM lands on a half-disposed context. Mirror the SIGHUP cleanup pattern (the file already removes the SIGHUP listener on the disposal path — see the existing `:55` block) so SIGTERM is symmetrically removed.
- `packages/opencode/src/cli/cmd/tui/thread.ts:166` — adds `process.on("SIGTERM", () => stop())` **before `stop` is declared** (`const stop = async () => { ... }` lives at `:170`). Because of TDZ this throws `ReferenceError: Cannot access 'stop' before initialization` the first time SIGTERM is delivered between the listener registration and the closure capturing the binding. The `:174` `process.off("SIGTERM", stop)` works only because by then `stop` is in scope. Move the listener registration **below** the `const stop = ...` declaration, or hoist `stop` to a `function` declaration.
- `packages/opencode/src/cli/cmd/tui/thread.ts:174` — `process.off("SIGTERM", stop)` passes the bare `stop`, but the `on()` at `:166` registered `() => stop()` (an anonymous wrapper). **Listener identity does not match**, so `off()` is a silent no-op and the SIGTERM handler leaks across reload cycles. Either register `stop` directly (no arrow wrapper) or store the wrapper in a named `const`.
- `packages/opencode/src/project/instance.ts:19-21` — `createLruCache<string, Promise<InstanceContext>>({ maxEntries: 20 })` swap is the right call. Two questions the diff doesn't answer: (1) **what's the eviction callback?** Each `InstanceContext` likely owns runtime resources (subscriptions, FS watchers, the `WorkspaceContext`); evicting an entry that backs a still-running session will crash that session at next access. The PR needs an `onEvict` that either rejects the eviction when the entry is in-use or runs the same teardown path as explicit removal. (2) Why 20? On a multi-project Linux daemon scenario this can churn; please cite the worst-case observed working set or make it configurable via env.
- `packages/opencode/src/session/prompt.ts:1065-1068` — adding `Effect.ensuring(Effect.sync(() => controller.abort()))` alongside `Effect.onInterrupt` is exactly the right diagnosis: `onInterrupt` only fires on the interrupt path, `ensuring` runs in **all** termination paths (success, failure, interrupt) and is the documented way to release resources. This is the single most impactful hunk in the PR and resolves the 37-GB heap growth claim. One nit: `controller.abort()` after a normal completion is harmless but emits a useless `AbortError` on any listener that observes signals; if `controller.abort` is observed downstream, gate it on `if (!controller.signal.aborted)`.

## Verdict

**request-changes**

## Rationale

The `prompt.ts` heap fix and the LRU cache are correct and high-value. But the SIGTERM plumbing in `thread.ts` has two real bugs in the same 8-line hunk: the TDZ reference and the listener-identity mismatch in `off()`. As shipped, the SIGTERM path that this PR is supposed to harden is itself either crashing on first invocation (TDZ) or leaking handlers across reloads (off mismatch). And the LRU swap needs an eviction story before it lands — silently evicting an in-use `InstanceContext` will turn an OOM crash into a confusing use-after-free. Fix the three handler issues, document the eviction policy (or wire one), and this becomes merge-as-is. It's the right set of changes scoped well; it just needs one more pass.
