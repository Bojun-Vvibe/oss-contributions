# google-gemini/gemini-cli #26366 — fix(sea): run forked helper scripts directly instead of spawning a new session

- PR: https://github.com/google-gemini/gemini-cli/pull/26366
- Author: noxymon
- Head SHA: `42b74eea86cf5bbbc1178d4daba7697fa0ddaea4`
- Updated: 2026-05-02T07:14:01Z
- Closes: #26365

## Summary
In the SEA (Single Executable Application) build, `child_process.fork(modulePath, args)` uses `process.execPath` as the Node interpreter — which is the SEA binary itself. Any `fork()` call (notably `@lydell/node-pty`'s `WindowsPtyAgent._getConsoleProcessList()` calling `conpty_console_list_agent.js`) therefore launches a second top-level CLI session instead of running the helper script. PR detects fork invocation in `sea/sea-launch.cjs` (via `typeof process.send === 'function'`), scans `argv` for the requested `.js` script, normalizes argv to `[binary, script, ...args]`, and loads the script via `Module.createRequire(scriptPath)(scriptPath)`.

## Observations
- `sea/sea-launch.cjs:228-270`: detection uses `typeof process.send === 'function'`. The PR description correctly notes that `NODE_CHANNEL_FD` is no longer reliable on modern Node. `process.send` is the right signal — it is set iff the parent passed `stdio: ['ipc', ...]` (which `child_process.fork` always does) and isn't truthy under any normal user invocation. Solid.
- `sea/sea-launch.cjs:240-244`: `path.resolve(raw)` before comparing against `process.execPath` is essential and the inline comment says so. Without absolute resolution, a relative argv entry would slip past the `=== process.execPath` check on the duplicate-execPath slot. Good.
- `sea/sea-launch.cjs:246-249`: the auto-`.js` append (`if (!scriptPath.endsWith('.js') && fs.existsSync(scriptPath + '.js'))`) is generous but reasonable — Node's `fork()` accepts module paths without extensions and resolves them, so SEA needs to mimic that. Worth adding: should it also try `.cjs` / `.mjs`? `node-pty`'s helper is `.js` so this is fine for the immediate bug, but a comment noting the intentional limitation would prevent confused follow-up bug reports for `.mjs`-shaped helpers.
- `sea/sea-launch.cjs:253`: `nodeModule.createRequire(scriptPath)` — confirm `nodeModule` is `require('node:module')` somewhere earlier in the file (the diff snippet doesn't show that import). Per Node docs, `createRequire` requires an *absolute* path or `file://` URL; the PR resolves to absolute via `path.resolve(...)` above, good.
- `sea/sea-launch.cjs:262-266`: silent `return` on script-load failure with the explicit comment *Don't fall through to a gemini session — that is the bug we are preventing* is the right choice. The fork() parent's IPC timeout will recover. Without this guard, a botched helper would do exactly the regression the PR is trying to prevent.
- `sea/sea-launch.fork.integration.test.cjs` is a real end-to-end harness — spawns a tiny IPC-aware helper through the SEA binary and asserts a 30 ms PASS vs. the BEFORE 15 s timeout. Good repro fixture; CI should wire this in or it will rot. The PR doesn't show CI changes, so confirm whether `npm test` / Windows CI invokes it, or it sits as a manual repro.
- **Out-of-scope diffs in this PR**: the diff also includes an unrelated `integration-tests/ui-hang-repro.test.ts` (new file) and a substantial rewrite of `packages/cli/src/ui/contexts/KeypressContext.tsx` (paste-batching with a `PASTE_BATCH_THRESHOLD = 32`, ESC-sequence state machine across data chunks). Those changes are the right idea (they fix a real and separate UI hang on large clipboard pastes) but they are *not* mentioned in the PR title, body, or "Related Issues". This is the most important review note: **either split the keypress/paste change into its own PR, or expand the title + body to cover both fixes explicitly so reviewers (and `git blame` / changelog tooling) can find them.** Reviewers will look for the SEA fork bug and miss that they are also evaluating a parser change in the keystroke hot-path.
- Specifically on the bundled keypress change: the `inEscape` / `escapeIntro` state needs to survive across data chunks (it does, declared outside the returned closure). The OSC final-byte detection (`ch === '\x07' || ch === '\\'`) only checks for the trailing `\\` of an `ESC \` ST sequence, not the `ESC` itself — that means a pathological OSC ending with a literal backslash would early-exit. In practice OSC is BEL-terminated so this is a non-issue, but worth a comment.
- `PASTE_BATCH_THRESHOLD = 32` is a magic number; even a one-line comment justifying *why 32 vs 16 vs 64* (presumably "above this, per-character React renders dominate; below this, fast typing must still produce keypress events") would help future tuning.

## Verdict
`request-changes`

The SEA fork() fix itself is high-quality and ready. The PR scope and title need to be honest about the bundled keypress/paste rewrite — either split into two PRs or update the description so the second change is reviewable as such. Merging in current shape would silently land a parser change in a hot path under a misleading title.
