# Review: sst/opencode PR #25547 — feat(server): native HttpApi listener with Bun.serve + WS upgrade

- **Head SHA:** `a9c5d07e9ff8f6da6fbb2aadcc6c575d098386fd`
- **State:** OPEN
- **Size:** +353 / -0, 2 files
- **Verdict:** `needs-discussion`

## Summary
Bun-only proof-of-concept listener at
`packages/opencode/src/server/httpapi-listener.ts:1-244` that drives the
experimental HttpApi web handler directly (no Hono in the middle) and
intercepts the PTY connect WS path inline via `Bun.serve` + `server.upgrade()`.
The Hono path remains intact — this is staging code for the eventual Hono
deletion.

## Strengths
- Tight scope: 244 lines of listener + 109 lines of test, no production wiring.
  `Server.listen()` is intentionally not rewired (correctly noted in the PR
  body and code comments).
- The `ptyConnectPattern` regex at `httpapi-listener.ts:43` is derived from
  `PtyPaths.connect` instead of being hard-coded — keeps it in sync with route
  literal moves.
- The `WithInstance.provide({ directory })` bridge in
  `httpapi-listener.ts:158-166` correctly addresses the AsyncLocalStorage gap
  the author flagged: `Pty.connect` requires `LocalContext`, not just
  AppRuntime services. The directory is captured at `fetch` time and
  passed via `server.upgrade(req, { data })`, which is the only context that
  survives into `websocket.open`.
- Tests at `test/server/httpapi-listener.test.ts:1-109` exercise both HTTP
  parity (`/pty/shells`) and end-to-end PTY echo, skipping on Windows like the
  existing PTY suite.

## Concerns / discussion points

1. **Workspace-proxy WS is a stubbed TODO.** The PR explicitly defers
   workspace-proxy WS forwarding (see the long TODO at
   `httpapi-listener.ts:118-126`). Until that lands, this listener cannot
   replace the Hono backend for any deployment that uses workspace proxying.
   Worth confirming the rollout sequence: does the Node adapter come first,
   or workspace-proxy WS? Both are "biggest unknown" candidates.

2. **`webHandler()` is module-level memoized.** The author notes that
   `.dispose()` permanently invalidates the singleton for every other caller.
   `stop()` here intentionally does not dispose — fine — but multiple
   `listen()` calls in the same process will share state. Should the listener
   refuse to start twice, or document the singleton constraint at the export
   site?

3. **`parseCursor` accepts `-1`.** `httpapi-listener.ts:48-52` allows
   `parsed >= -1`. Confirm this matches the existing Hono-path semantics —
   the existing PTY route should be the source of truth for sentinel values.

4. **`asAdapter` swallows send/close errors silently.**
   `httpapi-listener.ts:54-75` wraps every operation in `try/catch` with
   `// ignore`. For close-after-close that's correct, but a noisy debug log
   would help diagnose bridge bugs in the WS upgrade flow.

5. **Idle timeout of `0` (`httpapi-listener.ts:99`)** disables Bun's idle
   socket reaping. Verify this is intentional for long-lived PTY sessions
   and that the workspace-proxy follow-up will not regress this.

## Verdict rationale
The architecture is sound and the author's probe-and-document approach is
exemplary (see the "Surprises in the upgrade API" section in the PR body).
But this PR is fundamentally a design checkpoint — merging it as-is adds
dead code paths that won't be exercised in production. Discuss the Hono
deletion sequencing before merging, and consider gating the new module
behind an experimental flag with an opt-in test mode.
