# QwenLM/qwen-code#3242 — fix(cli): preserve startup keystrokes through full UI init

- **PR**: https://github.com/QwenLM/qwen-code/pull/3242
- **Author**: @xxih
- **Head SHA**: `dc61cbf`
- **Base**: main
- **State**: OPEN
- **Scope**: medium — 12 files (+625/-41). Two new utility modules (`earlyInput.ts`, `startupProfiler.ts`), keypress-context replay support, init wiring, three test files.

## Summary

Fixes a real "fast typists lose characters during CLI startup" bug (links upstream issue #3224). Root cause: between the moment the binary starts and the moment the React/Ink REPL mounts and registers its keypress handlers, anything the user types into the TTY hits the raw stdin stream and is silently discarded — Node's `process.stdin` is in default flowing mode but no listener is attached yet, so data events fire into the void. Fix: add an `earlyInput.ts` module that, at process entry (`packages/cli/index.ts` and `packages/cli/src/gemini.tsx`), immediately attaches a buffered listener to stdin in raw mode, captures every byte, and exposes `consumeBufferedInput()` for the REPL to drain once `KeypressContext` mounts. The replay path injects the buffered keystrokes into the `KeypressContext` queue before unblocking interactive input, so the user's pre-mount typing shows up exactly as if the REPL had been ready from the start.

Also adds a small `startupProfiler.ts` utility (independent of the input fix) to time-stamp init phases so future regressions in this window are observable.

## Diff anchors

- `packages/cli/src/utils/earlyInput.ts` — the load-bearing new module. Pattern: on import, set raw mode, attach `data` listener, append to an internal `Buffer[]`, expose `consumeBufferedInput(): Buffer[]` that drains and detaches. Right shape:
  - **Idempotent**: guards against double-init via a module-level `installed` flag, so importing from both `index.ts` and `gemini.tsx` is safe.
  - **Restoration**: tracks the original raw-mode state on install and restores it on consume — so if the REPL never mounts (early failure path), the TTY isn't left in raw mode after process exit. Verify the install path handles non-TTY stdin (piped input, CI) without throwing.
  - **No echo**: raw mode disables OS-level echo, so the user sees nothing during the buffering window. This is correct for matching the post-mount behavior (Ink also runs raw + handles its own rendering), but worth noting in the doc comment so future contributors don't "fix" the missing echo.
- `packages/cli/index.ts` — wires `earlyInput` install to the very top of the entry script, before any heavy imports. The position matters: any module-level `await import(...)` between the binary entry and the install call widens the lossy window. Verify the install actually runs before the first `await import` of the React/Ink subtree.
- `packages/cli/src/gemini.tsx` — secondary install site for the `gemini` subcommand path. The idempotency guard makes this safe.
- `packages/cli/src/ui/contexts/KeypressContext.tsx` — adds replay support. On mount, calls `consumeBufferedInput()` and re-emits each buffered byte through the same `useKeypress` dispatch path as live input. Critical detail: the replay must happen *after* downstream consumers have subscribed (otherwise the replay events fire into the same void the original keystrokes did) and *before* the input prompt unblocks. Confirm the replay site is `useEffect(() => { ... }, [])` (mount only), not a render-phase call.
- `packages/cli/src/ui/contexts/KeypressContext.replay.test.tsx` — the right test for this fix: seeds the early-input buffer with a known byte sequence, mounts the context with a mock subscriber, asserts the subscriber received the bytes in order. Pinning the *order* invariant matters because a Promise-based dispatch could reorder against live input mid-replay.
- `packages/cli/src/utils/earlyInput.test.ts` — covers install/consume/restore lifecycle and the idempotency guard.
- `packages/cli/src/utils/startupProfiler.ts` + tests — independent observability primitive. Useful, but conceptually a separate change. Worth splitting into its own PR if review velocity matters; bundling here is forgivable because the input-loss bug is exactly the kind of thing the profiler will help detect regressions in.
- `packages/cli/src/utils/relaunch.ts` and `sandbox.ts` — small touch-ups to thread the early-input lifecycle through the relaunch (sandbox-respawn) and sandbox-init paths. Without these, a re-exec for sandbox setup would lose the carefully-buffered keystrokes anyway. Right scope.
- `packages/cli/src/gemini.test.tsx` — adds wiring tests for the new lifecycle.
- `docs/users/configuration/settings.md` — minor docs touch.

## What I'd push back on

1. **Confirm non-TTY behavior.** `earlyInput` install in raw mode on a piped stdin (CI, `echo foo | qwen ...`) will fail or behave oddly. The install path should check `process.stdin.isTTY` and no-op cleanly otherwise. Add an explicit test for the piped-stdin case.
2. **Bracketed-paste sequences in the buffer.** If a user pastes during the lossy window and the terminal emits bracketed-paste markers (`\x1b[200~ … \x1b[201~`), the buffered bytes will replay through `KeypressContext`, which may not be in bracketed-paste mode yet. Verify the replay path handles or strips paste markers; otherwise the user sees the literal escape codes in their input.
3. **`startupProfiler` should be opt-in via env var** to keep the hot path zero-cost when not profiling. Confirm it's gated.
4. **Race on `consumeBufferedInput`.** If the install detaches its listener inside `consume`, any byte that arrives between the last buffered byte and the listener-detach is lost. Consider draining synchronously then atomically swapping the listener over to the live `KeypressContext` dispatch in the same tick. Worth a comment naming the gap if it can't be closed.
5. **Cross-platform: Windows ConPTY.** Raw mode on ConPTY has historically had quirks around `Ctrl+C` handling and key-event vs character-event modes. Confirm CI covers the Windows path; otherwise this fix may regress Ctrl+C signaling during the buffered window on Windows terminals.

## Verdict

**merge-after-nits** — the bug is real and well-known to fast typists, the fix is the right shape (capture early, replay on mount, restore TTY on early exit, idempotent install), and the test coverage targets the load-bearing invariants (order, idempotency, lifecycle). Block on the non-TTY no-op test and the bracketed-paste handling note before merge; the rest are nits.

Repo coverage: QwenLM/qwen-code (CLI startup input lifecycle — early stdin capture + replay through KeypressContext to fix lost keystrokes during React/Ink REPL init).
