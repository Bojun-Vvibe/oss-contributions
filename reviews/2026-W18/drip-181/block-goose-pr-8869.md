# block/goose#8869 — fix: correct WSL2 OS detection by removing PWD-based Windows override

- **Repo**: block/goose
- **PR**: [#8869](https://github.com/block/goose/pull/8869)
- **Head SHA**: `2efc795775247a6f2ffc4514aa010593c0d801a8`
- **Author**: JheisonMB
- **Diff stats**: +4 / −31 (1 file: `download_cli.sh`)

## What it does

The legacy `download_cli.sh` installer script had a tangled WSL2 detection
ladder that, when invoked from a Windows-mounted directory like
`/mnt/c/Users/foo/`, would decide the user wanted the **Windows** binary
even though the script was running inside a Linux WSL2 kernel. Result:
users who happened to `cd /mnt/c/...` and ran the installer got a
Windows `.exe` dropped at `$USERPROFILE/goose` and a confusing fallback
behavior. Fix: collapse the WSL detection to a single rule — "WSL is a
Linux kernel; install the Linux binary regardless of `$PWD`."

## Code observations

- `download_cli.sh:59-63` — pre-fix had three branches for the install
  directory (native Windows, WSL `/proc/version`-detected, WSL
  `$PWD =~ /mnt/[a-zA-Z]/`). Now collapses to two: native Windows
  (`$WINDIR`/`msys`/`cygwin`) and "everything else uses
  `$HOME/.local/bin`". Comment "Linux, macOS, and WSL all use the same
  bin directory" is exactly right.
- `:97-99` — pre-fix WSL block had a nested 17-line ladder around
  `$PWD =~ ^/mnt/[a-zA-Z]/`, `command -v powershell.exe`,
  `command -v cmd.exe`, and `-d "/c"`/`-d "/d"`/`-d "/e"` directory
  probes, all attempting to guess "is the user really on Windows even
  though they're in WSL?". Replaced with a one-line invariant:
  `OS="linux"`. Comment is the right framing: "WSL is a Linux
  environment regardless of the current working directory. The PWD
  (e.g. /mnt/c/) does not change the kernel — always install Linux."
- The remaining `elif [[ "$PWD" =~ ^/[a-zA-Z]/ ]] && [[ -d "/c" || -d
  "/d" || -d "/e" ]]; then OS="windows"; fi` branch survives at the
  bottom of the chain. That's the Git Bash on native Windows path
  (where `/c`/`/d`/`/e` are the Git Bash mount points). Correct to
  keep — it's a *non*-WSL case.

## Behavior change / risk

- **Real backward-compat shift**: a WSL2 user who previously ran the
  installer from `/mnt/c/...` and *intentionally* got a Windows
  binary will now silently get a Linux binary instead. That's the
  right behavior — running Linux binaries inside WSL2 is the
  supported and performant path — but a user who wired the resulting
  `goose.exe` into a Windows-side workflow (Task Scheduler, Windows
  shortcut) will be broken on next reinstall. Worth a release-note
  line.
- **No CI-level test**. The script has no test harness in the diff
  visible here. A small bats / shellcheck-driven matrix covering
  `OSTYPE` × `WINDIR` × `/proc/version` shapes would lock in the new
  invariant. Without it, the next maintainer who "helpfully" reverts
  one branch of the deletion (a real risk given how recently the
  `/mnt/[a-zA-Z]/` ladder was added) won't be caught.
- **Removed branches that may have served a purpose**: the
  `command -v powershell.exe` probe and the `-d "/c"` directory
  probes are gone from the WSL block but stay in the Git-Bash
  fallback. Confirm via grep that no other installer entry point
  depends on the removed branches; removal here doesn't help if
  another script duplicates the logic with the same bug.

## Verdict: `merge-after-nits`

The fix is the right semantic correction. WSL2 *is* Linux; treating
it as Windows because `$PWD` happens to be on `/mnt/c` was always a
heuristic that papered over a deeper question users shouldn't have
to answer at install time. Nits:

1. Add a release-note line: "WSL2 installs now always select the
   Linux binary regardless of working directory. Users who
   previously installed the Windows binary via `download_cli.sh`
   from a `/mnt/c/...` working directory will see the Linux binary
   on next install. To install the Windows binary, run the installer
   from a native Windows shell (PowerShell / Git Bash on Windows)."
2. Add a 30-line bats test (or shellcheck-checked smoke test) that
   sources `download_cli.sh` with the OS-detection block factored
   into a function, then asserts the function's return value across
   the matrix `(WINDIR set | OSTYPE=msys | OSTYPE=cygwin |
   /proc/version contains "Microsoft" | macOS | Linux | Git Bash)`.
   Without a test, the deletion will eventually be re-litigated by
   someone reading old issue threads.
3. Audit `download.ps1` (if it exists) and any other install entry
   point to confirm the same WSL-as-Windows heuristic isn't
   duplicated there. The bug class is "we tried to be too clever
   about cross-environment detection"; if it's in one installer
   it's likely in a sibling.
4. Add a `--target-os` / `--target-arch` CLI flag to
   `download_cli.sh` for users who genuinely need to override the
   detection (e.g. building a Windows install image from a Linux
   CI runner). That's the principled escape hatch instead of
   re-introducing PWD heuristics.

## Follow-ups

- Confirm with the docs site that any "install on WSL2" guidance
  doesn't say "make sure you `cd` somewhere outside `/mnt/c` before
  installing" — that was the old workaround for this bug, and it
  should be deleted.
- Consider whether the parallel `OS="windows"` branch at the
  bottom of the chain (`elif [[ "$PWD" =~ ^/[a-zA-Z]/ ]] && [[ -d
  "/c" || ... ]]`) should also be tightened to require `OSTYPE` to
  be `msys`/`cygwin` — the directory-existence probe alone could
  false-positive in unusual mount layouts.
