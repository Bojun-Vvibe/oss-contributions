# sst/opencode PR #24136 — feat(desktop): add support for Open in Windows Terminal in the header menu

- **Author:** NazmusSayad
- **Head SHA:** 1305ca55c9db69e8e78acabcc5c4ee5ab0e7823f
- **Files:** `packages/app/src/components/session/session-header.tsx` (+7 / −0),
  `packages/desktop-electron/src/main/ipc.ts` (+5 / −0),
  `packages/desktop-electron/src/renderer/index.tsx` (+5 / −1),
  `packages/desktop/src-tauri/src/lib.rs` (+12 / −1),
  `packages/desktop/src-tauri/src/os/windows.rs` (+21 / −0),
  `packages/ui/src/assets/icons/app/windows-terminal.svg` (+89),
  `packages/ui/src/components/app-icon.tsx` (+2 / −0),
  `packages/ui/src/components/app-icons/types.ts` (+1 / −0)
- **Verdict:** `merge-after-nits`

## What the diff does

Wires `wt.exe` (Windows Terminal) into the desktop "Open in…" menu as
a first-class entry, parallel to the existing PowerShell handling.

Specifically:
- `session-header.tsx` lines 39–46 + 85–95: adds `windows-terminal` to
  the `OPEN_APPS` tuple and the `WINDOWS_APPS` config with
  `openWith: "wt.exe"`.
- `desktop-electron/src/main/ipc.ts` lines 145–150: short-circuits
  the generic `execFile` path on Windows when `app` is `wt`/`wt.exe`,
  invoking it as `execFile("wt.exe", ["-d", path], { cwd: path })`.
- `desktop-electron/src/renderer/index.tsx` lines 144–148: bypasses
  `resolveAppPath` for `wt`/`wt.exe`, since `wt.exe` lives in a
  WindowsApps shim that path-resolution typically can't find.
- `desktop/src-tauri/src/lib.rs` lines 181–198: adds a parallel
  `is_windows_terminal` check that hands off to a new
  `os::windows::open_in_windows_terminal(path)`.
- `desktop/src-tauri/src/os/windows.rs` lines 464–484: spawns
  `wt.exe` with `CREATE_NEW_CONSOLE`, normalising file → parent dir
  before passing as `-d`.
- Plus an icon SVG and the new icon registration.

## Review notes

- Good directory normalisation in `windows.rs` 466–474: file →
  parent → fallback to `current_dir`. The Electron path at
  `ipc.ts:147` does *not* do this normalisation — it passes
  `path` (which may be a file) as both `cwd` and the `-d` argument.
  `wt.exe -d <file>` will error out with "directory not found" or
  open the wrong working dir. The Electron handler should mirror
  the Tauri normalisation, ideally by extracting a small shared
  helper.
- `["wt", "wt.exe"].includes(app.toLowerCase())` in two places —
  pull this into a constant or helper so future case variations
  (`WT.exe` from a config) don't drift between the two callsites.
- The Electron handler uses `execFile("wt.exe", ...)` directly,
  which depends on `PATH` containing the WindowsApps shim. On some
  enterprise images that shim is stripped; consider falling back
  to the documented `%LOCALAPPDATA%\Microsoft\WindowsApps\wt.exe`
  path with a clear error if neither resolves.
- Trivial: `desktop/src-tauri/src/lib.rs` line 562 introduces an
  unrelated whitespace-only deletion (`-` blank line). Worth
  reverting to keep the diff focused.
- No tests added. Tauri side could at least unit-test the
  file-vs-dir normalisation in a `#[cfg(test)]` block; the
  WindowsApps shim is mockable via a trait.

## What I learned

`wt.exe` is unusual on Windows in that it is intentionally not on
the searchable filesystem PATH for security/sandbox reasons — it's
a Microsoft Store app exposed through a per-user `WindowsApps` shim.
Any "open in terminal" feature has to special-case it instead of
relying on generic app-resolution, which is exactly what this PR
discovered the hard way.
