# block/goose #8869 — fix: correct WSL2 OS detection by removing PWD-based Windows override

- PR: https://github.com/block/goose/pull/8869
- Head SHA: `2efc795775247a6f2ffc4514aa010593c0d801a8`
- Files: `download_cli.sh` (+4 / -31)
- Relates: #8672

## Citations

- `download_cli.sh:60-66` — old code had a 4-arm OS detection for the `DEFAULT_BIN_DIR` decision: native-Windows arm (`WINDIR` / `OSTYPE=msys|cygwin`) → `$USERPROFILE/goose`; WSL arm (`/proc/version` matches `Microsoft\|WSL`) → `$HOME/.local/bin`; mount-point arm (`PWD =~ ^/mnt/[a-zA-Z]/`) → `$HOME/.local/bin`; default arm → `$HOME/.local/bin`. The middle two arms reduce to the default. PR collapses to: native-Windows → `$USERPROFILE/goose`; everything else → `$HOME/.local/bin`. Comment updated to "Linux, macOS, and WSL all use the same bin directory."
- `download_cli.sh:96-105` — old `OS=...` resolution had a *seven-branch* WSL-specific tree under the `/proc/version` arm: WSL on `/mnt/c/...` → `OS=windows`; WSL with `powershell.exe`/`cmd.exe` available AND on Windows mount → `OS=windows`; WSL with Windows interop tools but not on mount → `OS=linux`; WSL with no Windows interop tools → `OS=linux`. Plus an outer-level `PWD =~ ^/mnt/[a-zA-Z]/` arm that re-set `OS=windows` for any non-WSL Bash with `/mnt/c/` cwd. Plus a separate arm at line 122 for "Windows executables present" → `OS=windows`. PR collapses the WSL arm to a single `OS=linux` with comment: "WSL is a Linux environment regardless of the current working directory. The PWD (e.g. /mnt/c/) does not change the kernel — always install Linux."
- `download_cli.sh:106-110` — the dangling `elif [[ "$PWD" =~ ^/mnt/[a-zA-Z]/ ]]; then OS="windows"; fi` (which would catch a *non-WSL* Bash sitting in a `/mnt/c/...` cwd, an essentially impossible state outside WSL) is also removed. The "Windows executables present" arm (`elif command -v powershell.exe ...`) at line 122 is also deleted: presence of `powershell.exe` on PATH outside Native Windows means you're in WSL, which now correctly resolves to Linux earlier in the chain.
- The Git Bash detection (`elif [[ "$PWD" =~ ^/[a-zA-Z]/ ]] && [[ -d "/c" || -d "/d" ]]`) survives — that case (`/c/Users/...` cwd with `/c` dir present) is genuinely Native Windows under Git Bash and correctly resolves to `OS=windows`.

## Verdict

`merge-as-is`

## Reasoning

This is a "pure deletion is the fix" PR with a correct mental model. The bug being fixed is straightforward: a developer working in WSL2, sitting in `/mnt/c/Users/foo/projects/bar/` (a Windows-mounted directory accessed from inside WSL), runs `curl ... | bash` to install goose. The old detection logic noticed the `/mnt/c/` PWD prefix, decided "this looks Windows-y," and downloaded `goose-x86_64-pc-windows-msvc.zip`. That zip extracts a `.exe` that the WSL2 Linux kernel cannot execute as a native binary — and even if Windows-interop runs it, the binary then tries to use Windows-shaped config paths (`AppData\Roaming\...`) which are inaccessible from the Linux side via standard XDG conventions. The user gets a `.exe` in `~/.local/bin/` and nothing works.

The fix is correct because the underlying premise of the deleted logic was wrong: the *kernel* WSL2 is running on (Linux 5.15+) is what determines which binary works, not the directory the install script happens to be invoked from. A developer can `cd /mnt/c/Users/foo` and they're still on a Linux kernel — the only thing they need is the Linux binary in a Linux-style path. The seven branches of WSL-disambiguation logic were trying to second-guess this with a heuristic that was structurally wrong: `command -v powershell.exe`, `[[ -d "/c" ]]`, `$PWD =~ ^/mnt/[a-zA-Z]/` — none of these signals tell you whether the *runtime kernel* is Linux or NT, only whether the user happens to have Windows interop reachable. WSL2 *always* has Windows interop reachable; that's the design.

What the PR gets right:

1. **Pure deletion, no replacement.** The 31-line removal isn't compensated by 31 lines of "smarter detection elsewhere." The author correctly identified that the deleted code was net-negative — its absence produces a *more correct* outcome on every WSL invocation, not a less-correct outcome that some other code now papers over.

2. **The Git Bash arm is preserved.** This is the case where the user runs Git Bash on actual Native Windows (no WSL kernel), the cwd looks like `/c/Users/foo`, and `/c` is a real directory because Git Bash mounts the C: drive there. That arm correctly resolves to `OS=windows` because the runtime is genuinely the NT kernel. Author distinguished "looks like a Windows path" (false positive) from "is actually Native Windows" (true positive) and kept only the latter.

3. **Minimal blast radius.** The change is in `download_cli.sh` only, which is the curl-pipe-bash installer. It doesn't touch the binary, doesn't touch goose's runtime config-path logic, doesn't change anything for users who already installed correctly. A WSL2 user who previously got the wrong `.exe` will, on next install attempt, get the right binary; everyone else is unaffected.

4. **The PR description does the right framing.** It explicitly enumerates the symptoms ("Download of goose-x86_64-pc-windows-msvc.zip instead of the Linux binary," "Installation of a .exe file into a Linux binary path," "Attempts to use Windows-specific configuration paths") and verifies the OS-detection branches manually, which is the right level of rigor for an installer-script change.

Why this is `merge-as-is` rather than `merge-after-nits`:

- The PR doesn't add a shellcheck or a test, but `download_cli.sh` is a curl-piped installer — adding test infrastructure would be a much larger change than the fix warrants. The author did note `bash -n download_cli.sh passes syntax check`, which is the appropriate level of verification.
- The deleted code was load-bearing for nobody — the WSL-on-/mnt/c case was the only path that produced a different result from the simplified logic, and that result was the bug. There's no "this used to handle case X, now it doesn't" risk.
- The carve-out for native Git Bash is preserved, which is the only adjacent edge case that could plausibly regress.

The one nit I'd flag for the maintainer (not blocking): the PR removes the `command -v powershell.exe || command -v cmd.exe` arm entirely, which means a user running on a *non-WSL non-NativeWindows* environment that happens to have `powershell.exe` on PATH (e.g., PowerShell Core installed on Linux) would now resolve to `OS=linux` rather than `OS=windows`. This is *correct* — PowerShell Core on Linux runs Linux binaries — but it's a behavior change worth noting. Test the install on a Linux box with `pwsh` installed before merging if that's a non-trivial user segment.

Net: a clean structural correction that deletes net-negative complexity and produces the right outcome on every codepath. Ship.
