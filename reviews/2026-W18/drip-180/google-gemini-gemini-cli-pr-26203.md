# google-gemini/gemini-cli#26203 — fix(core): quiet ripgrep fallback log

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26203
- **Author:** @haosenwang1018
- **Head SHA:** `09bf5e6e` (full: `09bf5e6e755b966938d96c64c4e37df69ef16b42`)
- **Size:** +27/-1 across 2 files
- **Closes:** #26193

## What this does

Demotes one log line in `packages/core/src/config/config.ts:3683` from
`debugLogger.warn(...)` to `debugLogger.debug(...)`:

```
- debugLogger.warn(`Ripgrep is not available. Falling back to GrepTool.`);
+ debugLogger.debug(
+   `Ripgrep is not available. Falling back to GrepTool.`,
+ );
```

That's the one-line fix. The other +24 LOC is in `config.test.ts` —
`it('should register GrepTool as a fallback when useRipgrep is true but it is
not available', ...)` (line 2262 region) and the sibling `... when canUseRipgrep
throws an error` test get spies on `debugLogger.warn` and `debugLogger.debug`,
asserting the message is now logged via `.debug`, not `.warn`.

The behavioral surface — `GrepTool` is registered as the fallback, the
`logRipgrepFallback` telemetry event still fires with the same `event.error`
payload — is explicitly preserved by the existing assertions, plus the new
`expect(warnSpy).not.toHaveBeenCalledWith(...)` confirms the warn-channel
silence.

## Why this is worth taking

Ripgrep absence on a host machine is not actionable to most users —
`GrepTool` is registered automatically and works fine. Logging this at
`warn` level was producing noise on every CLI startup for any user without
ripgrep installed (which, per the linked issue #26193, is the whole reported
class). `debug` is the correct level: still observable when someone
explicitly cranks the log verbosity to investigate "why is grep slow",
silent in normal operation.

The fact that `logRipgrepFallback(this, new RipgrepFallbackEvent(errorString))`
is **not** demoted is the right call — telemetry should still see this so
the team can track ripgrep adoption rates. Only the user-facing console
line changes.

## What I'd nudge

1. **Test name fidelity.** The two test descriptions still read "should
   register GrepTool as a fallback when useRipgrep is true but it is not
   available" — they don't mention the log-level expectation that's now the
   *primary* thing they exercise. After this PR they're really
   "...as a fallback at debug log level when ripgrep is unavailable". Worth
   a tiny rename for grep-ability, but not blocking.

2. **There's only one site.** Both `canUseRipgrep === false` and
   `canUseRipgrep` throwing now log via `debug`. The error path arguably
   should still surface a one-time, non-startup warning when
   `canUseRipgrep` actually *throws* (vs cleanly returning `false`),
   because a thrown error can mean a corrupted ripgrep install that the
   user *can* act on (`brew reinstall ripgrep`). The current PR collapses
   both into the same `debug` channel. Consider keeping `warn` for the
   `error.toString().length > 0` case and only demoting the
   `canUseRipgrep === false` clean-fall case. Not blocking — current PR is
   a strict improvement over the status quo — but worth a follow-up.

3. **Singular log message change.** Same message string is duplicated in
   the diff via the multi-line wrap (Prettier formatting). One-string
   constant exported from the module would let tests reference it instead
   of duplicating the literal in `expect(...).toHaveBeenCalledWith('Ripgrep
   is not available. Falling back to GrepTool.')`. Drive-by territory.

## Verdict

**`merge-after-nits`** — the change is correct, the test coverage is
strong (it asserts both "warn was *not* called" and "debug *was* called"
with the exact string), and it directly addresses the reported issue.
The only real call-out is whether to keep `warn` for the throw path so a
broken ripgrep install stays user-visible.

## Nits

- Consider keeping `warn` when `canUseRipgrep` throws (vs returns `false`).
- Extract the message string to a const and reference it from both
  `config.ts` and `config.test.ts`.
- Rename the affected test cases to mention the log-level expectation.
