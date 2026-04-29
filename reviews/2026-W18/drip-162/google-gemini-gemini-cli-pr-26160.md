# google-gemini/gemini-cli PR #26160 — fix(cli): implement signal forwarding in relaunch

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26160
- Head SHA: `14e2fd63d9`
- State: OPEN, +79/-1 across 2 files (fixes #25590)

## What it does

Closes a signal-orphaning bug in the parent-relaunch path. Pre-PR `relaunchAppInChildProcess` spawned a child node process and awaited its `close` event, but installed *no* signal handlers on the parent. So when systemd, a terminal, or `kill <parent-pid>` sent `SIGTERM`/`SIGINT`/`SIGHUP` to the parent, the parent died immediately and the child became an orphan reparented to PID 1, continuing to hold ports/locks/file handles indefinitely.

This PR registers handlers for `SIGTERM`, `SIGINT`, `SIGHUP`, and `SIGUSR2` on the parent at child-spawn time. Each handler `child.kill(sig)` (gated on `!child.killed` for idempotence). Handlers are removed on both `child.on('error', ...)` and `child.on('close', ...)` paths via a `removeForwarders()` cleanup closure. Per-signal `process.on(sig, handler)` is wrapped in `try/catch` to absorb the platform-not-supported case (Windows lacks `SIGHUP`, `SIGUSR2`).

## Specific reads

- `relaunch.ts:68-72` — signal list `['SIGTERM', 'SIGINT', 'SIGHUP', 'SIGUSR2']`. Notable inclusions:
  - `SIGUSR2` is included because nodemon and several debugger-attach flows use it to trigger reload — forwarding it to the child is the right call so a `nodemon`-managed parent doesn't leave a child stranded on restart.
  - Notable **exclusions**: `SIGQUIT` (default core-dump signal — debatable whether to forward, but if you don't, `kill -3` to the parent silently leaves the child running), `SIGPIPE` (you almost never want to forward this), `SIGCHLD` (would be incoherent — that's how the parent learns the child died). The list is reasonable but `SIGQUIT` is worth considering.
- `relaunch.ts:73-86` — handler registration:
  ```js
  signals.forEach((sig) => {
    const handler = () => {
      if (!child.killed) {
        child.kill(sig);
      }
    };
    signalHandlers.set(sig, handler);
    try { process.on(sig, handler); }
    catch (e) { /* swallow */ }
  });
  ```
  Note the `try/catch` swallows *all* registration errors silently (including a real bug where `process.on` would throw for non-Windows-related reasons). At minimum, a `console.warn` or a tracing log inside the catch would help debug "why aren't my signals being forwarded?" reports. The empty `// Some signals might not be supported on all platforms (e.g. Windows)` comment is descriptive but logs nothing.
- `relaunch.ts:88-95` — `removeForwarders` symmetrically iterates and calls `process.off(sig, handler)`, also with a swallowing `try/catch`. If `process.on` succeeded, `process.off` should never fail with the same handler reference, so the catch here is defensive-paranoid but harmless.
- `relaunch.ts:111-114` — `child.on('error', ...)` and `child.on('close', ...)` both call `removeForwarders()` before resolving/rejecting. Critical detail: **without these cleanup calls, every relaunch cycle leaks four signal listeners on `process`**, and Node's default `MaxListenersExceededWarning` would fire after ~3 relaunches. The cleanup is correctly placed.
  - **Subtle gap**: the `child.send({ type: 'admin-settings', ... })` block at `:99-101` (between handler-registration and the Promise constructor) can `throw` if the child's IPC channel isn't ready yet — and that throw bubbles past `relaunchAppInChildProcess`'s outer try (if any) without `removeForwarders()` being called. A `try/catch` around the `child.send` (or moving the send *after* the Promise constructor inside an `.on('spawn', ...)` listener) would close that leak window.
- `relaunch.ts:99-101` — also worth noting: the `child.send({ type: 'admin-settings', settings: latestAdminSettings })` happens **synchronously after spawn**, before the Node child IPC channel may be ready. On macOS this usually works; on Linux it's been known to ENOENT under load. Pre-existing code, not this PR's change to fix.
- **Race window** — between `spawn()` returning and the four `process.on(sig, handler)` calls completing, a signal arriving will follow Node's default disposition (terminate parent, orphan child). This window is microseconds and probably not worth mitigating, but worth knowing.
- **Process-group consideration**: a more robust design would `spawn(..., { detached: false })` (already the default) AND set the child's process group to the parent's, so that `kill -- -<parent-pgid>` reaches both atomically. But this PR's "explicit-forward" approach is simpler and correctly handles the common SIGTERM-from-systemd case.
- `relaunch.test.ts:297-336` — the new test is well-targeted:
  - Spawns a mocked child, captures `vi.spyOn(process, 'on')` calls.
  - Finds the SIGTERM handler in the captured calls and invokes it directly.
  - Asserts `child.kill` was called with `'SIGTERM'`.
  - Closes the child to resolve the promise.
  - Asserts `processOffSpy` was called with `'SIGTERM'` and the same handler reference (cleanup path).
  - Restores spies. Clean test shape. **Doesn't pin the negative case** (cleanup happens on `'error'` path too) and doesn't pin **all four signals** (only SIGTERM is exercised, not SIGINT/SIGHUP/SIGUSR2). The handlers are identical so coverage is fine, but a parameterized `it.each(signals)` would be near-zero-cost.
- The test's `'should forward signals to the child process'` only exercises one signal name. Worth a second test exercising the `child.killed` short-circuit (call the handler twice and assert `child.kill` is only called once).

## Risk

1. **Silently-swallowed registration errors** make this code hard to debug if it ever doesn't work as expected. A `tracing` or `console.warn` inside the catch is cheap and the right shape.
2. **`child.send` throw window** between spawn and the Promise constructor can leak handler refs. Wrap the send in try/catch with cleanup.
3. **No SIGQUIT** — debatable. If users `kill -3 <parent>` to dump and exit, the child currently survives.
4. **Test coverage** — only SIGTERM exercised, not all four.

## Verdict

`merge-after-nits` — fixes a real lifecycle bug with the right primitive (explicit `signal.forEach + child.kill(sig)`), correctly cleans up on both terminal events of the child (`error`, `close`). Three small nits: log-on-catch for diagnostic, `try/catch` around `child.send` to prevent leak, parameterize the test across all four signals.

## What I learned

Signal forwarding from a relaunching parent to its spawned child is exactly the kind of "obvious-once-you-see-it" UX bug that bites operators (their CI's `docker stop` sends SIGTERM, the parent dies cleanly, the child holds a port until the 30s SIGKILL, deploys are slow and confusing). The right shape is: register handlers at spawn-time, forward to child by name (don't hardcode SIGTERM since `kill -1` and `kill -10` matter too), clean up on every termination path of the child. The asymmetric coverage (SIGUSR2 included for nodemon, SIGQUIT excluded) is a defensible choice but should be commented; otherwise a future maintainer adds SIGQUIT "for completeness" and accidentally suppresses the dump behavior users rely on.
