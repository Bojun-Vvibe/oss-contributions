# PR #24866 — fix(mcp): stop oauth callback server

- **Repo:** sst/opencode
- **Link:** https://github.com/sst/opencode/pull/24866
- **Author:** Hona
- **State:** OPEN
- **Head SHA:** `24ad3e878d9fd5667d80100b676c8c74215ecdc4`
- **Files:** `packages/opencode/src/mcp/index.ts` (+71/-63)

## Context

The MCP OAuth flow uses a small local HTTP callback server (`McpOAuthCallback.ensureRunning(...)`) to receive the authorization code from the browser redirect. The callback server is started lazily on the first OAuth attempt but was never being shut down — so once OAuth completed, failed, was cancelled, or was even just re-attempted in a tight loop, the listener was left open and the process couldn't exit cleanly. The PR description's repro (1000 iterations of `ensureRunning(...)` ⇒ `isRunning()` returns `true` and the parent process hangs until killed at the 3 s command timeout) shows the leak as a process-lifetime bug, not a memory bug.

## What changed

A single named effect, `stopOAuthCallback = Effect.tryPromise(() => McpOAuthCallback.stop()).pipe(Effect.ignore)`, is hoisted at `mcp/index.ts:244` and then attached at three lifecycle points:

1. **Service finalizer** — added at the bottom of the existing `Layer.scoped` cleanup block right after `pendingOAuthTransports.clear()` (`mcp/index.ts:533`). So when the `mcp` layer is torn down (process shutdown, instance reload), the listener goes with it.
2. **`authenticate` happy + sad paths** — the entire body is wrapped in an inner `Effect.gen(...)` whose result is piped through `.pipe(Effect.ensuring(stopOAuthCallback))` (`mcp/index.ts:863`). `Effect.ensuring` is the right primitive here because it runs on success, failure, *and* interruption — which is what you want for a one-shot OAuth dance.
3. **`finishAuth`** — same inner-gen + `Effect.ensuring(stopOAuthCallback)` pattern (`mcp/index.ts:894`). Belt-and-suspenders against the case where `finishAuth` is reached via an external API path that bypassed `authenticate`.
4. **`removeAuth`** — explicit `yield* stopOAuthCallback` after `pendingOAuthTransports.delete(mcpName)` (`mcp/index.ts:903`). When credentials are revoked, any in-flight OAuth listener for that server is now also torn down.

## Design analysis

This is the right shape. Three observations:

1. **`Effect.ensuring` over `acquireRelease`.** The author chose `ensuring` instead of `acquireRelease`/`scoped` because the listener's lifetime isn't *strictly* one-call — it's process-lived and refcount-shaped (multiple in-flight OAuth flows can share the listener until the last one finishes). `ensuring` is the conservative choice: it guarantees stop *after* this particular `authenticate`/`finishAuth` resolves, even if it's the last user, but still tolerates the listener being already-stopped (`Effect.ignore` swallows the no-op error). If `McpOAuthCallback.stop()` is internally idempotent (it should be), this is fine. Worth confirming in `McpOAuthCallback.stop` that calling it twice in quick succession from two parallel `authenticate` finalizers doesn't produce a confusing log line.

2. **Concurrent OAuth flows.** If two MCP servers initiate OAuth back-to-back (rare but legal), the first one's `Effect.ensuring(stopOAuthCallback)` will fire the moment its `finishAuth` returns — potentially closing the listener while server #2's browser redirect is still in flight. The PR doesn't address this. Either (a) `ensureRunning` is refcounted and `stop` only actually stops at refcount 0, or (b) the listener will be re-created by server #2's `ensureRunning` and the redirect will be lost. The repro proves stop-then-restart works, but a comment near `stopOAuthCallback`'s definition explaining the assumed invariant ("stop is safe at any time; ensureRunning will reanimate") would future-proof this.

3. **Wrap pattern produces large diff for small change.** The `+71/-63` is mostly indentation churn from wrapping `Effect.fn(... function* () { body })` into `Effect.fn(... function* () { return yield* Effect.gen(function* () { body }).pipe(Effect.ensuring(...)) })`. The semantic delta is: 4 callsite additions of `stopOAuthCallback`, one definition. Reviewers should diff with `--ignore-all-space` to see the real change.

## Risks

- **Idempotency of `McpOAuthCallback.stop`.** If `stop` throws on already-stopped, the `Effect.ignore` swallows it silently — fine for correctness but masks a future regression where `stop` starts throwing for a different reason (e.g. socket-close error). Consider downgrading from `Effect.ignore` to `Effect.catchAll(err => log.debug("stop oauth callback failed", { err }))` so the warning is at least visible at debug level.
- **Long-running CLI sessions.** If a user runs `opencode` interactively and re-authenticates multiple MCP servers over an hour, each `authenticate` will start-then-stop the listener. That's a port-bind-then-release cycle on every flow. Cheap on Linux, occasionally flaky on Windows under aggressive port reuse. Probably fine, but worth noting in the test plan.

## Verdict

**Verdict:** merge-after-nits

Right fix, right primitive (`Effect.ensuring`), right test repro. Two nits: (1) document the `ensureRunning`/`stop` refcount/idempotency assumption near `stopOAuthCallback`'s definition; (2) consider `Effect.catchAll(log.debug)` instead of `Effect.ignore` so future stop-side errors aren't silently dropped.

---

*Reviewed by drip-156.*
