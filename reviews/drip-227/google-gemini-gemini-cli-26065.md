---
pr: google-gemini/gemini-cli#26065
title: "fix(cli): ensure sandbox proxy cleanup and remove handler leaks"
head_sha: 11be845fc0aa5dcd52d14ecb621627f762ecd4e3
verdict: merge-after-nits
drip: drip-227
---

## Context

`start_sandbox` in `packages/cli/src/utils/sandbox.ts` registers
`process.on('exit'|'SIGINT'|'SIGTERM', stopProxy)` to tear down the
proxy subprocess / container on shutdown. Two latent bugs: (a) the
`stopProxy` closures were declared inside the per-branch sandbox
blocks, so the `finally` block at function exit had no way to invoke
them on the normal-exit path — proxy processes outlived the CLI on
`Ctrl-D` clean exits; (b) handlers were never removed, so re-running
`start_sandbox` in the same Node process (tests; future programmatic
use) accumulated stale handlers on the global `process` emitter. Plus
the `process.kill(-pid, 'SIGTERM')` and `execSync('docker rm -f ...')`
calls would crash the parent if the proxy had already died.

## Diff walkthrough

The fix lifts `stopProxy` to function scope at `sandbox.ts:58`:

```ts
let stopProxy: (() => void) | undefined = undefined;
```

Both branches now assign to the outer binding instead of shadowing
with `const`. The Docker-network branch at `:193-203` and the
container-network branch at `:752-762` both wrap their kill primitives
in `try/catch { /* ignore */ }`. The container branch also migrates
from `execSync(\`${command} rm -f ${SANDBOX_PROXY_NAME}\`)` to
`spawnSync(command, ['rm', '-f', SANDBOX_PROXY_NAME], { stdio: 'ignore'
})` — important because `execSync` runs through a shell and would
inject `SANDBOX_PROXY_NAME` unquoted; `spawnSync` with an args array
is safer.

The redundant `process.off('exit', stopProxy)` immediately before each
`process.on(...)` call is deleted at `:204-208` and `:763-767` — those
`off` calls were no-ops because the variable was a fresh closure on
each entry, and they're now correctly performed in the `finally`
block at `:815-820`:

```ts
} finally {
  if (stopProxy) {
    stopProxy();
    process.off('exit', stopProxy);
    process.off('SIGINT', stopProxy);
    process.off('SIGTERM', stopProxy);
  }
  patcher.cleanup();
}
```

Order matters: `stopProxy()` runs first (best-effort cleanup of the
proxy), then `process.off(...)` removes the listeners regardless of
whether `stopProxy()` threw — the inner try/catch guards against that.

## Tests

`sandbox.test.ts:722-779` adds `should register and unregister proxy
exit handlers`. It spies on `process.on` and `process.off`, mocks the
`docker images` and `docker run` spawn calls, drives `start_sandbox`
to completion, and asserts the symmetric pair:

```ts
expect(onSpy).toHaveBeenCalledWith('exit', expect.any(Function));
// ... SIGINT, SIGTERM
expect(offSpy).toHaveBeenCalledWith('exit', expect.any(Function));
// ... SIGINT, SIGTERM
```

The symmetric `on`/`off` count pin is exactly the right shape for
catching a future regression where someone "simplifies" the `finally`
block.

## Risks / nits

- The test asserts `expect.any(Function)` rather than `the same
  function reference`. A future bug that registered with one closure
  and unregistered with a different one would still pass. Consider
  capturing the listener from the `on` mock and asserting `off` was
  called with the same reference.
- The `try/catch {}` blocks swallow all errors silently. A `debugLogger`
  call in the catch arm would help diagnose proxy-cleanup failures.
- Only the Docker-container branch is exercised by the new test; the
  Docker-network branch at `:193-203` is structurally identical but
  uncovered.

## Verdict

**merge-after-nits** — correct fix for two real leaks (process
handlers + proxy lifetime), with a regression test that pins the
symmetric on/off contract. Tighten the test reference assertion and
log inside the try/catch arms.
