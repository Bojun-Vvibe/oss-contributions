---
pr: 8869
repo: block/goose
sha: 2efc795775247a6f2ffc4514aa010593c0d801a8
verdict: merge-as-is
date: 2026-04-28
---

# block/goose #8869 — fix: correct WSL2 OS detection by removing PWD-based Windows override

- **Author**: JheisonMB (Jheison Martinez Bolivar)
- **Head SHA**: `2efc795775247a6f2ffc4514aa010593c0d801a8`
- **Size**: 4 added / 31 deleted in 1 file (`download_cli.sh`).
- **Relates to**: #8672

## Scope

Closes a real "WSL2 user gets a Windows .exe installed into a Linux bin
directory" install bug. `download_cli.sh` was using `$PWD =~ ^/mnt/[a-zA-Z]/`
as a signal that the user "wanted the Windows build" even though
`uname -s` correctly reported `Linux`. Result on a developer working from
a Windows-mounted drive (e.g. `cd /mnt/c/users/me/proj && curl ... | bash`):
download of `goose-x86_64-pc-windows-msvc.zip`, install of a `.exe`
binary into `$HOME/.local/bin/`, and downstream config attempts against
Windows `AppData\Roaming` paths that don't exist on the Linux side.

## Specific findings

- `download_cli.sh:62-64` — `DEFAULT_BIN_DIR` selection: the previous
  three-branch chain (`WINDIR`/`OSTYPE` → `USERPROFILE/goose`; WSL via
  `/proc/version` → `~/.local/bin`; `/mnt/[a-z]/` PWD → `~/.local/bin`;
  default → `~/.local/bin`) collapses to a two-branch chain (Windows native
  → `USERPROFILE/goose`; everything else → `~/.local/bin`). Correct: the
  three "default to `~/.local/bin`" arms were behaviorally identical; the
  PWD branch was only there to mirror the `OS` detection logic below, and
  `OS` no longer needs it either.
- `download_cli.sh:96-100` — the WSL detection branch: was a 13-line
  nested conditional that flipped to `windows` on PWD-under-`/mnt/` plus
  some best-effort `powershell.exe`/`cmd.exe` interop checks. Now collapses
  to a 3-line block:
  ```
  elif [[ -f "/proc/version" ]] && grep -q "<WSL-kernel-marker>\|WSL" /proc/version 2>/dev/null; then
    # WSL is a Linux environment regardless of the current working directory.
    # The PWD (e.g. /mnt/c/) does not change the kernel — always install Linux.
    OS="linux"
  ```
  This is the correct semantic — `uname -s` and the kernel are what matter
  for which binary the runtime needs, not the user's `cd` location. The
  comment correctly names the invariant.
- `download_cli.sh:101-103` — also removed: the standalone `elif [[ "$PWD"
  =~ ^/mnt/[a-zA-Z]/ ]]; then OS="windows"` branch (PWD detection outside
  the `/proc/version` check). This branch could only be reached on a
  non-Linux non-Windows-cmd-line system that happened to have a `/mnt/c`
  path — extremely unlikely false positive, deserves to be deleted.
- `download_cli.sh:104-106` — the `command -v powershell.exe ||
  command -v cmd.exe → OS=windows` heuristic is removed. Correct removal:
  WSL itself ships `powershell.exe`/`cmd.exe` in the default PATH (via
  Windows interop), so this heuristic was a pure WSL-false-positive
  generator the moment WSL with interop became standard.
- `download_cli.sh:107-109` — preserved: the
  `[[ "$PWD" =~ ^/[a-zA-Z]/ ]] && [[ -d "/c" || -d "/d" || -d "/e" ]]`
  branch, which targets Git Bash on actual Windows. That heuristic is
  legitimately needed (Git Bash is a Windows runtime that wants the
  Windows binary) and isn't broken by the WSL fix. Keeping it is right.

## Risk

Low. The deleted branches were producing the documented bug; removing
them strictly improves correctness for WSL users. The remaining
detection chain is:
1. `WINDIR`/`OSTYPE=msys|cygwin` → Windows (native)
2. `/proc/version` matches the WSL-kernel marker → Linux
3. `OSTYPE=darwin*` → macOS
4. Git Bash heuristic (`/[a-z]/` PWD + `/c` or `/d` or `/e` exists) → Windows
5. Otherwise default

This is the right priority order: native-Windows-cmd-line first
(unambiguous), WSL second (unambiguous via `/proc/version`), macOS third,
Git Bash fourth (heuristic), default last. The only user case that loses
coverage is "developer on WSL who explicitly wanted the Windows binary
because they're going to copy it to `/mnt/c/Program Files/...` for
Windows-side use" — that user can still do `OS=windows ./download_cli.sh`
or download manually. Worth a note in the comment that the env var
override path exists.

The changed comment shape uses inline rationale ("PWD does not change
the kernel") which is exactly the right kind of comment to leave
behind — it answers the obvious "why not look at PWD?" question that
will arise when someone tries to re-add the heuristic.

## Verdict

**merge-as-is** — net 27-line deletion of a heuristic that was
producing the documented bug, with the correct invariant comment
inline. The detection chain that remains is clean, prioritized
correctly, and matches the actual signals available to a shell
script (kernel via `/proc/version`, environment via `WINDIR`,
shell type via `OSTYPE`).

## What I learned

PWD-as-OS-detection is a recurring shell-script anti-pattern: the
working directory tells you nothing about what binary the kernel
can execute. Same family as "detecting the OS by looking at the path
separator" or "detecting Windows by checking if `\\` works in
strings." The right signals are always the ones that come from the
kernel/environment (`uname`, `/proc/*`, `OSTYPE`, env vars set by
the shell), never from anything user-controllable like the current
directory. This PR's commentary correctly internalizes that lesson.
