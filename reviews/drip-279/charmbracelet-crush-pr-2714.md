# charmbracelet/crush PR #2714 — update terminal notifier for macos

- URL: https://github.com/charmbracelet/crush/pull/2714
- Head SHA: `00a4144f42df9a62ed18e6a0f6e5d76dca88c392`
- Files touched: ~3 (README + new darwin file + likely glue)
- Verdict: **merge-after-nits**

## Summary
Adds a macOS-specific notification backend (`internal/ui/notification/native_darwin.go`)
that shells out to the optional `terminal-notifier` binary so notifications can
focus the originating terminal when clicked. Falls back to the existing `beeep`
backend when `terminal-notifier` is not installed. Updates README with brew
install instructions.

## Cited concerns

1. `internal/ui/notification/native_darwin.go:14-26` (new): the
   `terminalBundleIDs` map hardcodes a curated list of `TERM_PROGRAM` →
   bundle ID mappings (Apple_Terminal, ghostty, iTerm.app, WezTerm, kitty,
   Alacritty, vscode, plus tmux/screen sentinels). This will go stale —
   Warp, Tabby, Rio, and Hyper will silently fall back. Consider documenting
   the override path (env var?) so users can supply their own bundle ID
   without a code change, or at minimum log at debug level when a
   `TERM_PROGRAM` is encountered that's not in the map.

2. `internal/ui/notification/native_darwin.go` uses `os/exec` to invoke
   `terminal-notifier`. Couple of safety asks:
   - Confirm the binary path is resolved via `exec.LookPath` (not from a
     user-controlled string) — terminal-notifier args include the bundle ID
     and notification body, both of which can contain shell metacharacters
     or be attacker-influenced via tool output. `exec.Command` with separate
     args is fine; `exec.Command("sh","-c", …)` would not be. Confirm in
     the rest of the file.
   - Add a context-bounded timeout on the invocation; a hung
     `terminal-notifier` should not stall the UI.

3. tmux/screen entries map to `""`. Make sure the consumer treats empty
   bundle ID as "skip the focus-on-click feature, send a plain notification"
   rather than passing `-sender ""` to terminal-notifier (which the binary
   may reject).

4. README diff (`README.md:472-484`) is clear but worth noting that
   `terminal-notifier` is unmaintained upstream; consider linking to the
   recommended fork or adding a note about the macOS 14+ permission prompt.

## Verdict
**merge-after-nits** — nice UX improvement. Tighten the exec invocation
review and add an extension hook for unknown terminals before merging.
