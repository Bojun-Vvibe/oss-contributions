# sst/opencode#25962 — feat(desktop): move server to utilityProcess

- PR: https://github.com/sst/opencode/pull/25962
- Head SHA: `0bd7c18353dd568f1f1265c58c8e0a29cd44c20c`
- Size: +420/-71
- Verdict: **merge-after-nits** (man)

## Shape

Splits the desktop main process into two electron entry points by adding a second
rollup input at `packages/desktop/electron.vite.config.ts:40`
(`{ index: "src/main/index.ts", sidecar: "src/main/sidecar.ts" }`) so the
embedded local server moves out of the main electron process and into a
dedicated `utilityProcess` child. Main process now talks to the sidecar via
`process.parentPort`-style `MessagePort` IPC (declared at
`packages/desktop/src/main/env.d.ts:8-15`) instead of in-process function calls,
and gains a `SidecarListener` import from `./server` at `index.ts:48-55`.

## Notable observations

- The `tls` import at `index.ts:8` was widened from a named-import
  `{ getCACertificates, setDefaultCACertificates }` to namespace
  `import * as tls from "node:tls"`, then re-narrowed via a local cast type
  `NodeTlsWithSystemCertificates` at `index.ts:39-42`. This is a workaround for
  the system-cert APIs not being in `@types/node` for the bundled Node version
  yet — fine, but the cast loses the `"default" | "system"` enum at every call
  site. A one-line `// TODO: drop cast once @types/node ≥ 20.13` comment would
  make the workaround unmistakable.
- `drizzle` import is removed from `index.ts:66` — confirm it has been moved
  fully into the sidecar entry point and is not still imported anywhere in the
  main bundle (the diff truncates here, can't see `sidecar.ts` itself in the
  first 250 lines).
- `parentPort.on("message", listener)` typing in `env.d.ts:11` types `ports`
  as `unknown[]` — losing `MessagePortMain` typing means every IPC handler will
  need its own `as MessagePortMain` cast. Worth importing the electron type
  here even if it requires conditional declaration.

## Concerns / nits

- No test added covering "main crashes → sidecar still running" or the
  reverse — the whole point of moving to `utilityProcess` is fault isolation,
  so a smoke test that kills the sidecar pid and asserts main reports a
  `"sidecar-exited"` event would lock the contract.
- Migration story for users with existing in-process state (sqlite db handles
  open in main) is not in the PR body — the drizzle move implies the DB now
  lives in the sidecar, so any in-flight migrations during upgrade need to be
  thought through (covered in `setBackgroundColor`'s sibling code path?).
- The new `sidecar` rollup input doubles bundle output count; verify
  `electron-builder`'s `extraResources`/`asar` config picks up both bundles.
