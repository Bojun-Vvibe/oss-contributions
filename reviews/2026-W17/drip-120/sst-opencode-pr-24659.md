# PR #24659 — fix(tui): handle SIGINT and SIGTERM for graceful shutdown

- **Repo**: sst/opencode
- **PR**: #24659
- **Head SHA**: `e524a2c6`
- **Author**: rektide
- **Size**: +2 / -0 across 1 file
- **URL**: https://github.com/sst/opencode/pull/24659
- **Verdict**: **merge-after-nits**

## Summary

The TUI's `ExitProvider` already wires `process.on("SIGHUP", () =>
exit())` to run the registered teardown effects on terminal-driven
disconnect. This PR adds the same handler for `SIGINT` and `SIGTERM`
at `packages/opencode/src/cli/cmd/tui/context/exit.tsx:58-59`. The
intent is right: when something upstream sends `SIGTERM` (systemd,
Docker, a `kill <pid>`), we want the same on-exit cleanup that
`SIGHUP` already gets — flush sync, persist session state, release
the alt-screen, etc.

## Specific changes

- `packages/opencode/src/cli/cmd/tui/context/exit.tsx:58-59` — two
  new lines:
  ```ts
  process.on("SIGINT",  () => exit())
  process.on("SIGTERM", () => exit())
  ```
  registered alongside the existing `SIGHUP` handler in the
  `ExitProvider` provider effect.

## Risks

There are two real concerns the diff does not address, neither
fatal but both worth a follow-up commit before merge.

First, the TUI's input layer almost certainly already has a
`Ctrl-C` interactive handler that performs a soft-cancel (interrupt
the in-flight model turn, return to the prompt) rather than exit.
Bun delivers `SIGINT` to the process when the terminal is in cooked
mode and Ctrl-C is pressed. With this PR, the very first Ctrl-C will
now race against the input-layer handler and may force-exit the TUI
mid-turn rather than the established interactive cancel-then-exit
two-stroke flow. The author should either (a) confirm the Bun TUI
runtime swallows `SIGINT` while the alt-screen is active and only
delivers it on detach, or (b) gate the `SIGINT` registration on a
runtime check so interactive Ctrl-C still cancels rather than exits.

Second, the existing `exit()` returned by the provider effect is
called many times (one per signal). If `exit()` is not idempotent,
two near-simultaneous signals (e.g. `SIGTERM` from a parent at the
same time the user hits Ctrl-C) will run teardown twice. Worth a
one-line `if (exited) return` guard on the closure rather than
trusting every consumer to be reentrant-safe.

## Verdict rationale

The change is two lines and the underlying intent is correct, but
the `SIGINT`-vs-interactive-Ctrl-C overlap is a real UX regression
risk that warrants either a comment explaining why it's safe in this
codebase or a one-line carve-out. Merge after the author either
confirms the Ctrl-C handling story or adds the gating guard.
