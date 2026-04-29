# google-gemini/gemini-cli #25944 — fix(cli): harden terminal state recovery and keypress parsing robustness

- **PR:** https://github.com/google-gemini/gemini-cli/pull/25944
- **Head SHA:** 481e2ed6a9bc4752e27448fdcc847d0ed22998f4
- **Files changed:** 3 files, +141 / −119 (`AppContainer.tsx`, `AppContainer.test.tsx`,
  `KeypressContext.tsx`)
- **Verdict:** `merge-after-nits`

## What it does

Fixes the recurring "Ctrl+L stops working after a model response" class of bugs by
attacking two root causes:

1. **Terminal protocol re-enable on idle.** Adds a `useEffect` in `AppContainer.tsx:2178-2192`
   that, whenever `streamingState === StreamingState.Idle`, fires a 50ms-delayed
   `terminalCapabilityManager.enableSupportedModes()` + `setRawMode(true)` +
   `stdout.write('\x1b[?1004h')` (focus reporting). This restores Kitty / modifyOtherKeys
   / focus-events that ANSI RIS sequences from tool output silently reset.
2. **Parser timeout robustness.** In `KeypressContext.tsx:359-385` `createDataListener`
   now (a) accepts `Buffer` and converts to UTF-8 string up front so the
   for-of-iteration doesn't hand byte-numbers to the parser, and (b) tracks the
   timeout id more carefully so an in-flight `yield` waiting on the rest of an escape
   sequence eventually flushes instead of swallowing the next valid key.

Also wires `terminalCapabilityManager.enableSupportedModes()` into `refreshStatic`
(`AppContainer.tsx:666`) so Ctrl+L itself re-arms the terminal after clearing.

## What's good

- The Buffer → string conversion (`createDataListener` line ~363) is a real bug
  fix — `for (const char of buffer)` on a Node Buffer yields numbers, not chars,
  and the parser was silently dropping those frames.
- Test added: `'calls enableSupportedModes when refreshing static'`
  (`AppContainer.test.tsx:3143-3158`) pins the refreshStatic ↔ capability re-arm
  contract.

## Nits / risks

1. The 50ms `setTimeout` in `AppContainer.tsx:2185` is a magic number to "let Ink
   finish its frame". This works empirically but couples to Ink's render scheduler
   — if Ink ever moves to a microtask queue or a 60Hz frame this either wakes too
   early (and the re-arm gets clobbered) or too late (visible keypress lag on idle
   transitions). Comment the rationale + a fallback if `streamingState` flips back
   to non-Idle inside the 50ms window (the cleanup function does clear the timeout,
   good).
2. No test covers the `Buffer`-input path of `createDataListener` — the Buffer
   conversion bug is asserted only by the runtime fix, not by a regression test.
   Adding a one-liner that calls the listener with `Buffer.from('\x1b[A')` would
   pin it.
3. The unconditional `stdout.write('\x1b[?1004h')` is a write-without-restore: on
   shutdown nothing emits `\x1b[?1004l`. If gemini-cli already has a global teardown
   for focus reporting that's fine; if not, terminals that retain mode across
   processes (some tmux configs) will keep emitting `[I` / `[O` for the next
   foreground program. Worth a teardown audit.

## What I learned

"Idle" is a useful re-arm trigger for terminal capability managers — much better
than waiting for explicit user input — but the 50ms Ink-frame delay is the kind
of magic number that ages badly across Ink major versions. Same goes for any
write-only escape (`?1004h`) that has no symmetric restore on exit.
