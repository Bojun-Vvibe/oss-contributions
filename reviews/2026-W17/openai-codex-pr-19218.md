# openai/codex#19218 — macOS seatbelt flags for Mach services and Apple events

**What changed.** Adds three repeatable CLI flags to `codex sandbox macos` (and the legacy `codex debug seatbelt` alias):
- `--allow-mach-service SERVICE` — adds `mach-lookup` permission for a named global service.
- `--allow-appleevent-destination BUNDLE_ID` — adds `appleevent-send` to a specific bundle ID.
- `--allow-lsopen` — permits LaunchServices open APIs.

Threads the values through `RunCommandUnderSandboxParams` (a new struct that replaces a 7-positional-arg function — nice cleanup) into `CreateSeatbeltCommandArgsParams`, and emits scoped Seatbelt policy. README updated. Non-macOS builds keep the params via `#[cfg_attr(...)]` to avoid `unused` warnings.

**Why it matters.** Existing `--allow-unix-socket` already follows the same targeted opt-in shape; without these flags, debugging anything that wants to talk to a system daemon (notification center, AppleScript-driven tools, Finder integration) requires either disabling sandbox or `--full-auto`, both of which lose the entire scoped-debugging value proposition.

**Concerns.**
1. **`--allow-lsopen` is broad.** LaunchServices `open` can launch arbitrary apps and pass URLs/files to them — effectively a sandbox escape via "ask Finder to do it." This isn't wrong (the user opted in) but the flag deserves a stronger doc warning than "permits LaunchServices open APIs." Recommend mentioning explicitly that this allows launching arbitrary applications.
2. **Service name is not validated against an allowlist.** `parse_non_empty_string` only rejects empty strings. A typo (`com.apple.notificationcenter` vs `com.apple.notificationcenterui`) silently produces a no-op policy entry; the user will see a denial and not know whether it's their flag or a missing one. Worth at least logging the resolved policy lines at `--log-denials` time.
3. **Bundle ID format unchecked.** Same shape: any non-empty string accepted. A malformed bundle ID won't be rejected by the parser and may produce a malformed Seatbelt policy. Verify Seatbelt's policy compiler errors loudly rather than ignoring the line.
4. **Flag-set explosion.** Now 4 `--allow-*` flags plus `--full-auto` plus `--log-denials`. Worth considering a future `--allow=mach:com.apple.x,appleevent:com.apple.y,lsopen` compact form before this grows further.
5. **Param struct refactor is the real win** — kills the "7 positional bools" smell. Land this even just for that.

Targeted, additive, opt-in. Good shape.
