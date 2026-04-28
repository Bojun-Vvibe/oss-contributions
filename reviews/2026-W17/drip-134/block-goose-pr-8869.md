# block/goose #8869 — fix: correct WSL2 OS detection by removing PWD-based Windows override

- PR: https://github.com/block/goose/pull/8869
- Head SHA: `2efc795775247a6f2ffc4514aa010593c0d801a8`
- Files changed: 1 (`+4/-31`) — `download_cli.sh`
- Relates to: #8672

## Verdict: merge-as-is

## Rationale

- **Pure-deletion fix with the right invariant named in the comment.** `download_cli.sh:97-100` replaces a 27-line tangle of PWD-based heuristics — `[[ "$PWD" =~ ^/mnt/[a-zA-Z]/ ]]` checks, `command -v powershell.exe`/`cmd.exe` probes nested inside the WSL detection branch, and a fallback `[[ "$PWD" =~ ^/mnt/[a-zA-Z]/ ]]` outside the `/proc/version` check — with a single `OS="linux"` assignment when `/proc/version` matches the WSL kernel marker grep pattern. The new comment ("WSL is a Linux environment regardless of the current working directory. The PWD (e.g. /mnt/c/) does not change the kernel — always install Linux.") names the load-bearing invariant correctly: the kernel WSL is exposing is Linux, the binary that runs is a Linux ELF, the file paths it reads are Linux, and which directory the user happened to `cd` into is irrelevant to which build to download. This is exactly the right framing.
- **Symptoms and fix scope match cleanly.** PR body lists three concrete failure modes — downloading `goose-x86_64-pc-windows-msvc.zip` (a `.zip` of a Windows `.exe`), installing into `$HOME/.local/bin/` (a Linux-style binary path), and attempts to use `AppData\Roaming` for config — all three are downstream consequences of the same upstream `OS="windows"` misclassification, and the fix at the `OS=` assignment site eliminates all three with no further downstream changes needed. That's the correct surgical scope.
- **The `DEFAULT_BIN_DIR` simplification at `:62-65` is symmetric with the `OS=` simplification.** Previously the script had a separate WSL-specific branch that set `DEFAULT_BIN_DIR="$HOME/.local/bin"` — but Linux/macOS/WSL all use the same path, so the WSL branch was a no-op duplicate of the default. Collapsing it to one comment-explained branch is right.
- **Native Windows path is preserved.** The `[[ "${WINDIR:-}" ]] || [[ "${windir:-}" ]] || [[ "$OSTYPE" == "msys" ]] || [[ "$OSTYPE" == "cygwin" ]]` check at `:96` is unchanged and runs first, so Git Bash / Cygwin / native Windows-shell installs still correctly resolve to `OS="windows"` — only the WSL-impersonating-Windows path is removed. The "Git Bash with Windows-style mount points" cell at `:103-105` is also preserved as a separate elif branch — that's a real codepath (Git Bash exposes `/c/` etc. without a `/proc/version`) that this PR correctly leaves alone.
- **Removed `command -v powershell.exe || cmd.exe` heuristic at the bottom of the original block.** This was a particularly fragile signal — WSL2 ships with Windows interop that makes `powershell.exe`/`cmd.exe` callable from Linux processes by default, so this heuristic *always* fired for any normal WSL2 install and contributed to the bug. Removing it is correct.

## Nits / follow-ups

- A one-line follow-up to add `bash -n download_cli.sh` to CI (PR body mentions running it manually) would prevent future regressions in the same shell-script-only file.
- Worth a one-line note in the script's leading comment that "WSL2 is always Linux" is the canonical assumption — to discourage a future contributor from re-adding the PWD heuristic next time someone wants to install the Windows build from inside WSL (the answer is: launch a real Windows shell, not WSL).
