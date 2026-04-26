# sst/opencode #24412 — fix: Buffer stdin before prompt UI appears

- **Repo**: sst/opencode
- **PR**: #24412
- **Author**: DSteve595 (Steven Schoen)
- **Head SHA**: cc8229e3e9da78679a9e334c3b590a9ddc6114cc
- **Base**: dev
- **Size**: +224 / −13 across 7 files; bulk in new
  `startup-input-buffer.ts` (+80) and two new tests (+31, +73).

## What it changes

Adds a `createStartupInputBuffer()` factory wired through `app.tsx:122`
that captures stdin written before the prompt UI is mounted. The
buffer is plumbed into `PromptRefProvider` (`prompt.tsx:7-22`) and
drained by `Home`'s prompt-binding callback (`home.tsx:30-46`) in a
priority order:

1. `route.prompt` (deep-link / restored state)
2. `args.prompt` (CLI `-p`)
3. `startupInputBuffer.drain()` (stdin captured pre-mount)

Buffer is disposed both on bind-time drain and via
`onBeforeExit → startupInputBuffer.dispose()` in `app.tsx:131`.

## Strengths

- Real UX bug: users who type immediately after launching the TUI
  used to lose those keystrokes because stdin wasn't being read until
  React mounted the input component. Capturing them in a side buffer
  is the correct fix.
- Drain logic is idempotent via `startupDrained` flag in
  `prompt.tsx:9` — calling `drainStartupInputBuffer` twice is safe,
  and the second call returns `false` so callers can branch on it.
- Doesn't override an already-typed input: `if (!input || ref.current.input) return false`
  in `prompt.tsx:21` ensures buffered stdin only seeds an *empty*
  prompt — nice safety net for fast typers who manage to type into
  the actual UI before the drain hook fires.
- Two test files: a unit test for the buffer itself (+31) and a
  precedence test (+73) verifying the route/args/buffer ordering.
  Good shape.

## Concerns / asks

- The `bind` callback in `home.tsx:30` is triggered on each render
  where `setRef` fires — relying on `once` (a closure variable) to
  prevent double-application is fragile across HMR / fast-refresh.
  Worth a comment explaining why a ref + closure is used instead of
  `useEffect`.
- `startupInputBuffer.dispose()` is called inside both `drain` (via
  `prompt.tsx:20`) and `onBeforeExit`. Dispose-on-already-disposed
  needs to be a no-op; if it throws, the exit path will fail. Worth
  an idempotence test.
- The buffer's lifecycle starts at `tui()` entry (line 122) — if a
  user invokes a non-tui code path that also reads stdin, the buffer
  may swallow input meant for that path. Not a regression (current
  flow always goes through `tui()`), but a follow-up tracking issue
  for "future non-TUI commands" would be prudent.
- 80-line `startup-input-buffer.ts` not visible in this slice — the
  verdict trusts that it correctly removes its `process.stdin` data
  listener on `dispose()` (failure to do so leaks a listener and may
  steal input from later child processes).

## Verdict

`merge-after-nits` — solid fix for a real UX gap with correct
test coverage. Asks: a dispose-idempotence test and a comment on
the `once` closure pattern.
