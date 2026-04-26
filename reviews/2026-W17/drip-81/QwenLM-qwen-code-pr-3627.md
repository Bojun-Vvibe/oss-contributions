# QwenLM/qwen-code PR #3627 — feat: add macOS desktop app installer

- **PR:** https://github.com/QwenLM/qwen-code/pull/3627
- **Author:** huangrichao2020
- **Head SHA:** `7df9f2970b4e`
- **Stats:** +387 / -0 across 5 files
- **Verdict:** `needs-discussion`

## What this changes

Adds an opt-in macOS desktop app installer for the `qwen` CLI. The deliverable has three pieces: a 222-line `scripts/installation/install-qwen-macos-app.sh` that builds an AppleScript-backed `.app` bundle wrapping `tell application "Terminal" → do script "qwen"`, a 148-line `docs/users/installation/macos-desktop-app.md` walkthrough, a 14-line README addition with a curl-pipe-bash one-liner, a `_meta.ts` nav entry, and a binary `qwen-icon.png` shipped in-tree at `scripts/installation/qwen-icon.png`. After install, the user gets a Spotlight/Launchpad entry that opens Terminal.app and runs `qwen`.

## Why I think this needs discussion

The packaging approach is the bare minimum that *technically* works on macOS, and it sidesteps every interesting question that a project at qwen-code's scale should answer before shipping a desktop entry point. Specifically:

**The `.app` is a wrapper script around an `osacompile` AppleScript, not a real macOS application.** That means: no code signing, no notarization, no entitlements, no Hardened Runtime, no auto-update path, no version embedded in `Info.plist` (so the Finder shows "Qwen Code" forever even after upgrading the underlying CLI), no quarantine handling for the curl-pipe-bash install path (Gatekeeper will warn the first time the user double-clicks if the script ran without the user being attached to a TTY in some flows), and no graceful failure when Terminal.app is not the user's preferred terminal. The doc at `macos-desktop-app.md:163-167` acknowledges the last point ("Terminal.app is your default terminal emulator") but doesn't suggest iTerm2/Ghostty/Alacritty alternatives or detect the configured default. This will generate Spotlight-launches-into-the-wrong-terminal support tickets indefinitely.

**Two of the three install paths run remote shell scripts as root-adjacent operations.** The README addition at line 17 promotes `bash -c "$(curl -fsSL https://raw.githubusercontent.com/QwenLM/qwen-code/main/scripts/installation/install-qwen-macos-app.sh)"` — i.e., curl-pipe-bash from a `main` branch URL with no checksum, no GPG signature, and no version pin. The script then downloads `https://raw.githubusercontent.com/.../qwen-icon.png` at install time (line 352 of the install script), giving a second curl-without-verification fan-out. Worse, the script's preamble at lines 207-214 will `exec bash` if invoked under a non-bash shell, which means the user-typed `bash -c "$(curl ...)"` line works under zsh by re-execing — but also masks the case where a downloaded copy was tampered with at the shell-detection layer. For a project that bills itself as developer tooling, the install story should at minimum publish a SHA256 alongside the script and have the README pin to a tag, not `main`.

**The `osacompile` AppleScript embeds `do script "qwen"` literally, which bakes in the assumption that `qwen` is on the user's `PATH` *as Terminal.app sees it*.** When Terminal.app spawns a new shell for the AppleScript, it runs as a login shell and sources `~/.zprofile`/`~/.bash_profile` but not `~/.zshrc`/`~/.bashrc`. Anyone who installed `qwen` via npm under a user-installed Node (nvm/fnm/volta) will get "qwen: command not found" on first launch, and the troubleshooting doc at `macos-desktop-app.md:162-170` only mentions "Make sure your shell profile loads nvm/fnm" as a one-liner — that's a multi-day debugging session for a non-expert user who clicked the icon expecting it to work. A real fix is to either (a) probe `command -v qwen` *during install* and embed the absolute path into the AppleScript, or (b) use a `.command` file with an explicit `#!/usr/bin/env -S bash -l` shebang.

**Bundling the binary `qwen-icon.png` in-tree.** Every clone of the repo now carries the icon bytes, and every install path (local script vs. curl) has to special-case whether it's available locally vs. needs to download. The conditional at install-script lines 348-355 already handles this, but it would be cleaner to either (a) host the icon at a stable CDN URL and always download it, or (b) keep it in a release artifact instead of the source tree.

## What I'd want to see before flipping to merge-after-nits

A maintainer-written design note covering: signing/notarization plan (or explicit "we don't sign — accept the Gatekeeper warning"), how the install path can be tag-pinned, what happens when `qwen` isn't on Terminal.app's login-shell `PATH`, and whether the `.app` should be published as a release artifact rather than built locally. Without that, this is a personal helper script that happens to live in the upstream repo — and once it's there, every macOS user-support ticket for the CLI will start with "did you use the desktop installer?".

## Bottom line

The intent is fine and the script is competently written, but the packaging shape commits the project to indefinitely supporting an unsigned, unversioned, terminal-emulator-coupled launcher with two curl-without-verification install paths. Convert to a discussion / RFC issue before merging.
