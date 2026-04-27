# google-gemini/gemini-cli #25958 — fix(cli): implement signal forwarding in relaunchAppInChildProcess

- **Repo**: google-gemini/gemini-cli
- **PR**: #25958
- **Author**: achernez
- **Head SHA**: 55605e7c8105001d1ebe003a294d9971138919cb
- **Size**: +85 / −2 across four files; load-bearing edits at
  `packages/cli/src/utils/relaunch.ts` (+37/−1) and
  `packages/cli/src/utils/relaunch.test.ts` (+42/−0).
- **Fixes**: #25590.

## What it changes

Three coordinated edits, two of them genuinely orthogonal to the headline
fix:

1. **Headline fix at `packages/cli/src/utils/relaunch.ts:65-114`.** Before
   this PR, `relaunchAppInChildProcess` spawned the child CLI via
   `spawn(...)` and then immediately set up `child.on('error', reject)` /
   `child.on('close', ...)` — no signal handling at all. So when systemd
   sent `SIGTERM` to the parent (e.g. on shutdown or `kill <pid>`), the
   parent died first and the child became reparented to PID 1, never
   receiving the signal. The fix adds at `:68-87` a
   `signalHandlers = new Map<NodeJS.Signals, () => void>()` over the
   list `['SIGTERM', 'SIGINT', 'SIGHUP', 'SIGUSR2']`, registering each
   handler via `process.on(sig, handler)` inside a `try/catch` (Windows
   doesn't support all of them). Each handler does
   `if (!child.killed) child.kill(sig)` — i.e. forwards the *same* signal
   to the child, preserving signal semantics for the SIGUSR2 reload case
   in particular. The matching `removeForwarders()` at `:96-103` is wired
   into both `child.on('error', ...)` (now `(err) => { removeForwarders();
   reject(err); }`) and `child.on('close', ...)` so the parent doesn't
   leak handlers across multiple relaunches.
2. **Bonus fix at `packages/vscode-ide-companion/package.json:114`**
   changing `"prepare": "npm run generate:notices"` to `"prepare": "node
   ./scripts/generate-notices.js"`. The `npm run` indirection was
   recursing into npm during install in some environments, breaking the
   prepare hook.
3. **Bonus debug logging at
   `packages/vscode-ide-companion/scripts/generate-notices.js:14-19`** —
   four `console.log` lines printing `projectRoot` / `packagePath` /
   `noticeFilePath` plus a `'NOTICES.txt generation started'` marker.

## Strengths

- **Right fix for a real bug.** Signal forwarding from a parent
  watcher/relauncher to its child is the documented pattern for any
  Node.js process supervisor (see PM2, foreman, supervisor.py source) —
  without it, `Ctrl+C` in the terminal kills only the parent (the
  controlling-terminal SIGINT goes to the foreground process group, but
  the child registered for the same group via spawn defaults receives
  SIGINT too in most cases — but `SIGTERM` and `SIGHUP` definitely don't
  propagate). The fix uses the right primitive (`process.on(sig, …)` +
  `child.kill(sig)`) and the right signal set
  (`SIGTERM`/`SIGINT`/`SIGHUP`/`SIGUSR2` covers daemon-shutdown,
  user-interrupt, controlling-terminal-disconnect, and Node's
  conventional reload/restart channel respectively).
- **Symmetric handler removal.** `removeForwarders()` at `:96-103` is
  invoked from both the error and close paths, so a sequence of
  relaunches doesn't accumulate handler subscriptions. The
  `try/catch` swallows removal errors, which is correct (removing a
  never-installed handler on Windows would otherwise throw).
- **Per-signal try/catch around `process.on(sig, handler)`** at `:80-84`
  correctly handles the Windows asymmetry: Windows Node treats
  `SIGHUP`/`SIGUSR2` as unsupported and `process.on` throws. The empty
  catch comment "Some signals might not be supported on all platforms
  (e.g. Windows)" names the contract explicitly.
- **The test at `relaunch.test.ts:297-336` pins the SIGTERM branch.**
  It uses `vi.spyOn(process, 'on')` to capture the registered handler
  by `(call) => call[0] === 'SIGTERM'`, invokes the handler manually,
  and asserts both `killSpy` was called with `'SIGTERM'` AND that
  `processOffSpy` removed the same handler reference after the
  child closed. The `spyOn(process, 'on')`/`spyOn(process, 'off')`
  cleanup at the end keeps the test from polluting the suite.
- **The `vscode-ide-companion/package.json` `prepare` fix** is a
  unrelated-but-justified one-line behavior fix that addresses a real
  install-time recursion. Bundling it in a "signal forwarding" PR is
  borderline scope-creep but defensible because both fixes touch the
  vscode-ide-companion package's child-process behavior.

## Concerns / nits

- **The pinned test only covers SIGTERM.** Three of the four signals in
  the list (`SIGINT`, `SIGHUP`, `SIGUSR2`) are untested. The failure
  mode for each is identical (the handler is the same closure shape), so
  a single test is *almost* enough — but a future contributor refactoring
  the loop into something subtly different (e.g. moving handler
  construction outside `forEach`) could break SIGUSR2 specifically and
  the test wouldn't catch it. A `it.each(['SIGTERM', 'SIGINT', 'SIGHUP',
  'SIGUSR2'])` parametrized version would cost two additional lines and
  pin all four branches.
- **No test for the Windows `try/catch` swallow.** The `try { process.on(sig,
  handler) } catch (e) {}` at `:80-84` exists for unsupported-signal
  cases on Windows, but there's no test asserting that an unsupported
  signal doesn't break the relaunch flow. A `vi.mocked(process.on)
  .mockImplementationOnce(() => { throw new Error('not supported'); })`
  test for one signal would lock that contract.
- **The handler `if (!child.killed) child.kill(sig)` at `:73-77` has a
  TOCTOU race:** between checking `child.killed` and calling
  `child.kill(sig)`, the child could exit on its own. In practice
  Node's `child.kill` is a no-op (or returns false) if the process is
  already dead, so this is harmless — but the guard is therefore
  redundant. Worth either removing the check (if Node's `child.kill`
  is genuinely safe on dead children, which it is per the docs) or
  adding a comment explaining what the guard is preventing.
- **`process.kill` vs `child.kill`.** The handler uses `child.kill(sig)`
  which sends to the immediate child PID only, not the child's process
  group. If the child has spawned its own children (e.g. an MCP server
  subprocess), those grandchildren won't receive the signal and will
  leak. For the gemini-cli relaunch scenario this is probably fine
  (the inner CLI does its own signal handling), but if it's not, the
  fix would need `process.kill(-child.pid, sig)` plus `detached: true`
  in the spawn options to send to the whole group. Worth a verification
  pass against the actual child's signal-handling behavior.
- **The bonus `generate-notices.js` debug logging at `:14-19`** is
  unconditional (`console.log`, not gated on `process.env.DEBUG` or
  similar) and ships in production. Four debug-prefixed lines on every
  invocation of `prepare` will pollute install logs across the
  ecosystem. Either gate behind `process.env.DEBUG_NOTICES` or remove
  before merge — the logs are clearly a "left in from debugging the
  prepare-hook fix" artifact.
- **The `prepare` script change** loses the `npm run`'s ability to
  pick up `package-lock.json`-resolved Node tooling (e.g. if a future
  `generate-notices.js` adds a `#!/usr/bin/env node` shebang and
  becomes directly executable, the script invocation form is fine, but
  if it acquires npm-script preconditions like `pre-generate:notices`
  hooks, those will silently stop running). Worth a one-line PR-body
  note explaining the regression in convention.
- **The promise-rejection contract changes silently.** Pre-fix, the
  promise resolved with `code ?? 1` on `'close'` and rejected only on
  `'error'`. Post-fix, the same — but the test at `:330` does
  `await expect(promise).rejects.toThrow('PROCESS_EXIT_CALLED')`,
  implying the relaunch path also calls `process.exit()` somewhere
  outside this snippet. Worth a one-line PR-body sentence explaining
  what the test's `PROCESS_EXIT_CALLED` is asserting against (likely
  a `process.exit` mock elsewhere in the test file's setup).

## Verdict

**merge-after-nits.** The signal-forwarding fix is correct, minimal,
right-sized, and pinned by a real handler-invocation test. The
bonus fixes are defensible but the unconditional debug logging in
`generate-notices.js` should be gated or removed. Before merge: (1)
gate or remove the four `console.log` lines in `generate-notices.js`,
(2) add parametrized `it.each` coverage for the three untested
signals, (3) verify the `child.kill` vs. process-group-kill behavior
matches the actual child's child-spawning patterns, and (4) drop the
redundant `if (!child.killed)` guard or comment its purpose.
