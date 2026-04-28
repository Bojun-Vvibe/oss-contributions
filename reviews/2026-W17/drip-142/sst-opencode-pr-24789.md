# sst/opencode#24789 — fix(opencode): prevent Warp Windows tab close on TUI exit

- **PR**: https://github.com/sst/opencode/pull/24789
- **Author**: @NicoAvanzDev
- **Head SHA**: `2057a13`
- **Base**: `dev`
- **State**: OPEN
- **Scope**: tiny — 2 files touched in `packages/opencode/src/cli/cmd/tui/`

## Summary

When opencode runs inside the Warp terminal on Windows, exiting the TUI was closing the entire Warp tab instead of returning to the shell. Root cause: opencode by default uses an alt-screen (xterm-style `?1049h/?1049l`) renderer; on Warp/Win32 the alt-screen leave sequence is interpreted at the host-emulator level as a tab-terminate signal because Warp on Windows uses ConPTY with a different alt-screen lifecycle than xterm/iTerm. Fix: detect `process.platform === "win32"` AND a Warp env signature (`TERM_PROGRAM` containing "warp", or `WARP_IS_LOCAL_SHELL_SESSION` / `WARP_SESSION_ID` set), and force the renderer into `screenMode: "main-screen"` for that combination, leaving every other terminal/OS combo on the existing default. Bundled with a small unrelated reorder of `e.preventDefault()` before `await exit()` in the prompt's `app_exit` keybind handler.

## Diff anchors

- `packages/opencode/src/cli/cmd/tui/app.tsx:9-13` — new `useMainScreenMode()` helper, correctly gated:
  - `if (process.platform !== "win32") return false` — short-circuits everything non-Windows so existing macOS/Linux behavior is byte-identical.
  - `if (process.env.TERM_PROGRAM?.toLowerCase().includes("warp")) return true` — primary discriminator (Warp sets `TERM_PROGRAM=WarpTerminal`).
  - belt-and-suspenders fallback `WARP_IS_LOCAL_SHELL_SESSION !== undefined || WARP_SESSION_ID !== undefined` — these are documented Warp-set vars and catch the case where TERM_PROGRAM is wiped by a wrapper script.
- `packages/opencode/src/cli/cmd/tui/app.tsx:20` — `screenMode: useMainScreenMode() ? "main-screen" : undefined` — `undefined` (not `"alt-screen"`) preserves the renderer's existing default code path; only the Warp+Win32 cell is changed. This is the right shape — additive, not a regression-shape switch.
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:31-35` — unrelated drive-by: swaps the order from `await exit(); e.preventDefault()` to `e.preventDefault(); await exit()`. The deleted comment "Don't preventDefault - let textarea potentially handle the event" is now contradicted, and the new ordering means the textarea is *guaranteed* not to see the keystroke even if `exit()` rejects. Probably correct (you don't want a half-exited app dispatching keys back to the textarea), but the rationale flip belongs in a separate commit and there's no test for either ordering.

## What I'd push back on (nits)

1. **`useMainScreenMode` is misnamed** — it's not a React hook (no `use*` semantics, no hook rules), it's a plain detection function called once during `rendererConfig`. Rename to `shouldUseMainScreen()` to avoid `react-hooks/rules-of-hooks` lint false-positives in the future.
2. **No test** — the failure mode is "Warp Windows tab terminates on exit," which is hard to e2e but easy to unit-test the predicate: feed the four env-var cells (Warp+win32, Warp+darwin, non-Warp+win32, non-Warp+darwin) and assert `shouldUseMainScreen` returns `true` only for Warp+win32. Without that, future env-var rename on Warp's side will silently break the carve-out.
3. **The prompt-handler change is unrelated** — split into its own PR or at minimum drop the misleading "let textarea potentially handle" comment with a one-line `// Always preventDefault first; once we begin exit() the textarea must not receive the keystroke.` rationale.
4. **`undefined` vs explicit `"alt-screen"`** — relying on the renderer's default to mean "alt-screen" couples this carve-out to the renderer's default never changing. Either pin it explicitly or add a comment naming the assumption.

## Verdict

**merge-after-nits** — the fix is correct shape (additive, gated, OS+terminal-double-discriminator), but ship the rename, add the 4-cell predicate test, and split the prompt-handler change into its own commit before merge.

Repo coverage: sst/opencode (TUI renderer carve-out for Warp/Win32 alt-screen interaction).
