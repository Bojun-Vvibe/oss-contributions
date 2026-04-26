# Review — charmbracelet/crush#2714: update terminal notifier for macos

- **Repo:** charmbracelet/crush
- **PR:** [#2714](https://github.com/charmbracelet/crush/pull/2714)
- **Author:** taigrr (Tai Groot)
- **Head SHA:** `00a4144f42df9a62ed18e6a0f6e5d76dca88c392`
- **Size:** +158 / −2 across 3 files (`README.md` +10/−2, `internal/ui/notification/native_darwin.go` +146/−0 new, `internal/ui/notification/native_other.go` +2/−0)
- **Verdict:** `merge-after-nits`

## Summary

Adds a darwin-only notification backend that, when `terminal-notifier` is installed via Homebrew, sends macOS desktop notifications with click-to-focus behavior targeting the originating terminal app. Detects the terminal via `TERM_PROGRAM` and maps to a hardcoded bundle-ID table at `internal/ui/notification/native_darwin.go:18-29`. Falls back to the existing `beeep.Notify` path when `terminal-notifier` is absent. Updates README to recommend `brew install terminal-notifier`.

## Technical assessment

The split between `native_darwin.go` (build tag `//go:build darwin`) and `native_other.go` (presumably `//go:build !darwin`, +2 lines as a stub) is correct Go pattern. The `terminalBundleIDs` map at `:18-29` covers Apple_Terminal, ghostty, iTerm.app, WezTerm, kitty, Alacritty, vscode, plus tmux/screen with empty strings as "runs inside another terminal" sentinels. That's a reasonable starter set for 90% of macOS terminal usage.

The `NativeBackend` struct at `:33-43` carries `terminalBundleID` (resolved at construction), `hasTerminalNotifier` (probed via PATH lookup, presumably), `icon any` (held for interface compat with the other backends, explicitly unused on darwin per the comment), and `notifyFunc` (defaults to `beeep.Notify` for fallback). `beeep.AppName = "Crush"` is set in `NewNativeBackend` at `:48` — that's a process-global mutation that's fine here because notification.NewNativeBackend is called once at startup, but worth noting that running multiple Crush instances in the same process would race.

The README update at lines `:472-485` is honest: documents that `disable_notifications: true` still works, `brew install terminal-notifier` enables click-to-focus, and explicitly notes the prior icon limitation. Good user-facing docs.

## Nits worth addressing pre-merge

1. **`tmux` and `screen` empty-bundle-ID handling needs the call-site check.** The map encodes `"tmux": ""` and `"screen": ""` as a way of saying "no bundle to target." The diff window doesn't show how `terminalBundleID == ""` is handled by the `Notify` path, but the right behavior is "fall back to no `-activate` flag on the `terminal-notifier` invocation, so the click is a no-op rather than crashing." Worth a one-line comment in the map plus a unit test that constructs `NativeBackend{terminalBundleID: ""}` and verifies the `terminal-notifier` argv does not include `-activate`.

2. **Bundle-ID table is incomplete and goes stale silently.** Missing: Warp (`dev.warp.Warp-Stable`), Tabby (`org.tabby`), Hyper (`co.zeit.hyper`), Rio (`com.raphaelamorim.rio`), Wave (`dev.commandline.waveterm`). When `TERM_PROGRAM` is unrecognized, fall back to `frontmostApplication` via osascript or just skip `-activate`. Also: `vscode`'s `TERM_PROGRAM` is actually `vscode` for the Code-OSS family; Cursor sets it to `vscode` too but the bundle is `com.todesktop.230313mzl4w4u92`. Document the heuristic limits.

3. **`ARG.terminalBundleIDs` is a package-level `var`.** That's a global; tests can't easily monkey-patch it for "what if user has Warp" coverage. Either make it `func defaultTerminalBundleIDs() map[string]string` or expose an injection seam on `NativeBackend`. Pure testability nit.

4. **`exec.Command("terminal-notifier", ...)` security note.** `terminal-notifier` is invoked with user-controlled title/message from agent output. macOS's `terminal-notifier` does its own argv handling but the `-message` flag value will be agent text, which can contain shell metacharacters. `exec.Command` itself doesn't shell-interpolate, so this is safe — but please add a one-line comment confirming "no shell interpretation, args passed as argv" so a future reader doesn't add `sh -c` wrapping.

5. **No CI for darwin path.** crush's CI matrix may or may not build the darwin tag; `internal/ui/notification/native_darwin.go` will only be compiled by GOOS=darwin runners. Confirm the CI matrix includes a `macos-latest` job that at minimum runs `go build ./internal/ui/notification` on this PR.

6. **`brew install terminal-notifier` is not actively maintained upstream.** The original `terminal-notifier` repo (`julienXX/terminal-notifier`) has not had a tagged release since 2019 and depends on deprecated NSUserNotification APIs (replaced by UserNotifications.framework in macOS 10.14). This isn't blocking — it works on current macOS — but a 12-month outlook should consider migrating to a tiny in-tree Swift helper or `osascript display notification`.

## Verdict rationale

`merge-after-nits` because the implementation is clean, the build-tag split is correct, and the README guidance is honest. Address the empty-bundle-ID handling (item 1) and CI matrix (item 5) before merge. The bundle-ID table completeness and the upstream `terminal-notifier` deprecation can be follow-ups.
