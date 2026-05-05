# sst/opencode #25837 — fix(desktop): trust system certificates

- Head SHA: `3be8861a5e08a733f8909023cc14ee0213884a20`
- Diff: +189 / −8 across 5 files (`packages/desktop/src/main/index.ts`, `packages/opencode/src/server/adapter.node.ts`, `package.json`, `bun.lock`)

## Verdict: `merge-after-nits`

## What it does

Makes the Electron desktop main process trust OS-installed CA certificates *in addition* to Node's bundled bundle, so corporate TLS-interception proxies (Zscaler, Netskope, Palo Alto, etc.) don't break server-side fetches the desktop app makes during config bootstrap. The mechanism is a one-time `setDefaultCACertificates([…default, …system])` call at startup against `node:tls`, gated by a try/catch so any failure logs and falls through to the Node default. To use that API the PR also bumps `@types/node` from `22.13.9` → `24.12.2` in `package.json` (matching the running Electron's Node, which already exposes `getCACertificates`/`setDefaultCACertificates` as of Node 22.x — they were stabilized later) and picks up an unrelated EventEmitter typing tweak in `packages/opencode/src/server/adapter.node.ts:8-26`.

## Why this is mostly fine

1. **The actual fix is six well-bounded lines.** `useSystemCertificates()` at `packages/desktop/src/main/index.ts:128-135` is called exactly once before `setupApp`, dedupes the union via `new Set([...])` (cert objects from `getCACertificates` are `string` PEM-encoded so set-dedup actually works), and on any error logs and returns — Node's default trust store stays in effect, so worst case is "no change in behavior" rather than "fewer trusted roots than before." Zero callers, zero hot-path cost.
2. **Placement is correct.** The call lives at module top level (line 69) before any `app.whenReady`/window creation, so by the time the embedded server starts making outbound HTTPS calls during `setupApp` (line 116-127) the trust store is already widened. Ordering matters here — putting it inside `setupApp` would race with the server's first fetch.
3. **The `adapter.node.ts` typing tweak is a real fix, not unrelated noise.** The new `@types/node@24` evidently tightens `ServerType.{on,off,once}` so the bare `server.off("error", fail)` calls at the old lines no longer typecheck against the union. Casting to `EventEmitter` once and calling `events.off(...)` / `events.once(...)` (`adapter.node.ts:11`, `:22-25`) is the minimal fix and keeps runtime semantics identical (`createAdaptorServer` returns a Node `Server` which *is* an `EventEmitter`).
4. **try/catch is the right shape.** `getCACertificates("system")` can throw on platforms where the OS keychain isn't reachable (locked-down Linux containers, missing Security framework on stripped-down macOS images). Catching and logging via `logger.warn(...)` keeps the desktop launch resilient.

## Nits

- **Log the count of certs added.** `logger.warn("failed to load system certificates", error)` only fires on failure. On success there's no breadcrumb to confirm the union actually picked up corporate roots, which is exactly what an operator debugging "the desktop app still can't reach the proxy" needs. A `logger.log("system certificates merged", { default: defaults.length, system: system.length, total: union.length })` between the two `getCACertificates` calls would pay for itself the first time someone files a "TLS handshake failure behind ZScaler" issue.
- **The `@types/node` 22→24 bump is across-the-monorepo via the workspace `package.json` override and propagates a long tail of resolution churn in `bun.lock`** (you can see the duplicated `@types/node@22.13.9` entries showing up under `bun-types`, `@slack/*`, `@types/express-serve-static-core`, etc., as second-level resolutions). That's not a *bug* — bun is correctly preserving sub-dep ranges — but reviewers should know this lockfile growth is a side-effect of one desired behavioral change in `packages/desktop`, not a separate dependency upgrade. A note in the PR body would help future bisects.
- **`getCACertificates("default")` already includes Node's bundled Mozilla store.** The `[...new Set([...defaults, ...system])]` union is correct, but it might also be worth pulling `"bundled"` explicitly if you want to guarantee the historical Mozilla bundle survives even when an admin has stripped the system keychain. Current behavior is fine; just flagging the surface.

## Citations

- `packages/desktop/src/main/index.ts:7` — new `node:tls` import
- `packages/desktop/src/main/index.ts:69` — `useSystemCertificates()` invocation site (pre-`setupApp`)
- `packages/desktop/src/main/index.ts:128-135` — `useSystemCertificates` body
- `packages/opencode/src/server/adapter.node.ts:11`, `:22-25` — `EventEmitter` cast for typing
- `package.json:42` — `@types/node` `22.13.9` → `24.12.2`
