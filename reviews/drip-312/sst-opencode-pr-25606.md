# sst/opencode PR #25606 — fix(tui): repaint Apple Terminal after resize

- URL: https://github.com/sst/opencode/pull/25606
- Head SHA: `fc24a76970d2dd6d66f80e7dba815f372b606d8e`
- Verdict: **merge-after-nits**

## Summary

Adds a targeted workaround for an Apple Terminal–specific bug where the
alternate screen is left partially stale after a `SIGWINCH` resize or screen
restore. The fix lives in `packages/opencode/src/cli/cmd/tui/app.tsx` as a
new `installAppleTerminalRepaintWorkaround(renderer)` helper that listens
to both `process.SIGWINCH` and the `CliRenderEvents.RESIZE` renderer event,
debounces them with a 30 ms `setTimeout`, and on fire either suspends/
resumes the renderer (the preferred path) or just emits
`\x1b[r\x1b[?6l\x1b[2J\x1b[H` and re-requests a render.

## Specific references

- `packages/opencode/src/cli/cmd/tui/app.tsx:1-14` — extends the
  `@opentui/core` import to bring in `CliRenderEvents`,
  `RendererControlState`, and the `CliRenderer` type. Reasonable; these
  are all used immediately in the new helper.
- `packages/opencode/src/cli/cmd/tui/app.tsx:101-145` — the new helper.
  Two correctness niceties worth noting:
  - The terminal probe is `process.env.TERM_PROGRAM !== "Apple_Terminal"`,
    so iTerm2/WezTerm/Ghostty/Kitty users are completely opted out by
    default. Good.
  - The escape preamble `\x1b[r\x1b[?6l\x1b[2J\x1b[H` (reset scroll
    region, disable origin mode, clear screen, home cursor) is the
    canonical "scrub the alt screen" sequence — fine.
- `packages/opencode/src/cli/cmd/tui/app.tsx:175-192` — wiring into the
  `tui()` lifecycle: a `removeAppleTerminalRepaintWorkaround` cleanup
  closure is captured at construction and invoked from `onExit`, which
  matches the existing `unguard` pattern.

## Commentary

The fix is conservative and the right shape: a single environment gate
(`TERM_PROGRAM=Apple_Terminal`), an explicit kill switch
(`OPENCODE_APPLE_TERMINAL_REPAINT_WORKAROUND=0`), and no behavior change
for any other terminal. The 30 ms debounce is reasonable for macOS resize
storms, and falling through to a plain clear when `suspend()` throws (or
when `controlState === EXPLICIT_SUSPENDED`) keeps the helper safe during
teardown.

Two nits worth raising before merge:

1. **`pending` is leaked on early-return paths.** When `repaint()` is
   called and `renderer.isDestroyed`, the function returns without
   clearing `pending`. That's harmless because `pending` is captured in
   the same closure and gets overwritten on the next `schedule()`, but
   it would be cleaner to set `pending = undefined` at the top of
   `repaint()` instead of inside the `setTimeout` body.

2. **No unit test or manual-repro note.** This is a terminal-specific
   visual regression; a paragraph in the PR body describing how to
   reproduce on Apple Terminal (or a screenshot) would let the next
   maintainer prove the fix still works after `@opentui/core` upgrades.
   Not a blocker — just a request.

3. **Cleanup return type.** `installAppleTerminalRepaintWorkaround`
   returns `(() => void) | undefined`, where `undefined` is implicitly
   returned by the early `process.env` gates. The caller already
   handles that with `?.()`. A tiny readability win would be to return
   a `noop` cleanup unconditionally so the type is just `() => void`.

None of these are correctness issues. The fix is well-scoped, opt-out-able,
and obviously safer than the prior behavior for affected users. Merge
after a quick polish pass.
