# charmbracelet/crush#2714 — update terminal notifier for macos

- **Repo:** charmbracelet/crush
- **Author:** taigrr (Tai Groot)
- **Head SHA:** `00a4144f` (00a4144f42df9a62ed18e6a0f6e5d76dca88c392)
- **Size:** +158 / -2

## Summary
Adds a `darwin` build-tagged `NativeBackend` that prefers `terminal-notifier`
when available, falling back to `beeep.Notify`. On click, it activates the
originating terminal by mapping `TERM_PROGRAM` to a bundle ID, with an
`osascript` lookup fallback. README updated to document the optional
`brew install terminal-notifier` step.

## Specific references
- `internal/ui/notification/native_darwin.go:1` @ `00a4144f` — new file, build-tagged `//go:build darwin`. Good isolation; no impact on linux/windows builds.
- `internal/ui/notification/native_darwin.go:13-23` @ `00a4144f` — `terminalBundleIDs` map hardcodes `Apple_Terminal`, `ghostty`, `iTerm.app`, `WezTerm`, `kitty`, `Alacritty`, `vscode`, plus empty entries for `tmux`/`screen` (correctly treated as inner-shell sentinels).
- `internal/ui/notification/native_darwin.go:75-87` @ `00a4144f` — `lookupBundleID` uses `osascript -e 'id of app "..."'` with a 2s `context.WithTimeout`. Reasonable upper bound; failures silently return `""` which falls through to `beeep`.
- `README.md:472-486` @ `00a4144f` — replaces the "lack icons" caveat with the new install hint.

## Observations
1. **Untrusted input into `osascript`**: `lookupBundleID` interpolates the raw `TERM_PROGRAM` env var into an AppleScript string with only `TrimSuffix(".app")` cleanup (`native_darwin.go:78`). An attacker controlling the parent process env could inject `"`/newlines and execute arbitrary AppleScript. Suggest validating against `^[A-Za-z0-9._ -]+$` before interpolation, or pass via stdin / `-l` literal.
2. **`hasTerminalNotifier` is checked once at construction**: if the user installs `terminal-notifier` after the agent starts, they won't benefit until restart. Minor — but the doc says "with `terminal-notifier` installed, clicking…", so a one-line mention that it's detected at startup would prevent confusion.
3. **No test for the bundle-ID map**: the table is the kind of thing that drifts (`vscode` vs `Code` is already a known fork divergence). A tiny table-driven test would prevent regressions.

## Verdict
**request-changes** — the `osascript` injection vector via `TERM_PROGRAM` should be patched before merge. Everything else is polish-level.
