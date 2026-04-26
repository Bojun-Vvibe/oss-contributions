# PR #24412 — fix: Buffer stdin before prompt UI appears

- **Repo**: sst/opencode
- **PR**: #24412
- **Head SHA**: `cc8229e3e9da78679a9e334c3b590a9ddc6114cc`
- **Author**: DSteve595
- **Size**: +224 / -13 across 7 files (incl. 2 new test files)
- **Verdict**: **merge-after-nits**

## Summary

Captures keystrokes typed during TUI startup (before the prompt
component is mounted) in a side-buffer and drains them into the
prompt's input field once the component binds. Without this, fast
typers lose the first few hundred milliseconds of input. New file
`startup-input-buffer.ts` parses raw stdin via `@opentui/core`'s
`StdinParser` to handle key events, paste blocks, backspace, and
Ctrl+U during the gap.

## Specific changes

- New `packages/opencode/src/cli/cmd/tui/startup-input-buffer.ts:8`
  — `createStartupInputBufferState()` builds an `StdinParser({
  armTimeouts: false })` and an empty input string. Synchronous
  parser is correct here: this thing doesn't own timers and is
  short-lived.
- `startup-input-buffer.ts:22-58` — `appendStartupInputBufferChunk`
  handles `paste` (decode bytes), `key` (filter to single-char
  printable, plus backspace/delete edits, plus Ctrl+U clear, plus
  swallow `return`). Notably, **Enter is silently dropped** — the
  comment says "should not submit anything" — this is the right
  call but needs a test.
- `app.tsx:121` — `createStartupInputBuffer()` is created right after
  `win32DisableProcessedInput()` and disposed in `onBeforeExit`.
- `context/prompt.tsx:7-25` — `PromptRefProvider` now takes
  `startupInputBuffer` and exposes `drainStartupInputBuffer(ref)`.
  Drain logic: `if (startupDrained || !ref) return false`, then
  set `startupDrained = true`, then dispose the buffer. Idempotent
  by design — good.
- `routes/home.tsx:31-46` and `routes/session/index.tsx:244-258`
  — both `bind` callbacks now have the same shape: seed explicit
  prompt first (route.prompt → args.prompt), then drain the startup
  buffer **only if `r.current.input` is still empty**. The "fill
  only if still empty" check at `prompt.tsx:23` is correct.

## Risks

1. **Parser race**: stdin is being read by *two* consumers between
   `tui()` start and prompt mount — the startup parser, then the
   real prompt parser. Need to confirm the startup buffer's
   `dispose()` actually unsubscribes its stdin listener so the
   second parser isn't getting a duplicated stream. Look for the
   `dispose` impl past line 80 of `startup-input-buffer.ts` (not
   shown in the head-200 cut).
2. **Ctrl+C / Ctrl+D**: the diff swallows `return` but doesn't
   appear to special-case Ctrl+C. If Ctrl+C arrives during the
   gap the user expects it to abort startup, not to be discarded.
   Worth verifying in the full file or via test.
3. **Test coverage** at `test/cli/cmd/tui/startup-input-buffer.test.ts`
   is 31 lines and `startup-prompt-precedence.test.tsx` is 73 —
   should cover: explicit prompt wins over buffered input,
   buffered input wins over empty state, Enter/Ctrl+U/backspace
   editing, paste merge.

## Verdict

`merge-after-nits` — the architecture (drain-once side buffer with
"only if empty" guard) is the right shape. Nits: confirm the
disposer detaches the stdin listener; add explicit Ctrl+C
handling; add an integration test for the explicit-prompt-wins
ordering.

## What I learned

A "fill the prompt with leftover startup input" feature is a
classic case where the *ordering rule* is the spec, not the
mechanism. The bind callbacks here encode that rule in plain
prose: "seed explicit prompt first; fill startup input only if
still empty." That sequence is the contract — the StdinParser is
just plumbing.
