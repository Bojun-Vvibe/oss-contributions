# sst/opencode#24678 — fix(desktop): disable in-app updater for non-AppImage Linux installs

- **PR**: https://github.com/sst/opencode/pull/24678
- **Author**: jeevan6996 (Jeevan Mohan Pawar)
- **Head SHA**: `c338facc6aa2e8962d38e8a21a47b7c9875245c2`
- **Verdict**: `merge-as-is`

## Context

The Electron desktop app's auto-updater module (electron-updater) only supports AppImage as a packaging format on Linux — for `.deb` / `.rpm` / Flatpak / Snap installs it either silently no-ops or, worse, attempts an update flow that ends in a confusing "cannot find update channel" error after the user clicks "Install". The previous gate at `packages/desktop-electron/src/main/constants.ts:10` was a flat `app.isPackaged && CHANNEL !== "dev"` — i.e. "if it's a packaged non-dev build, the updater is on" — which lit up the updater path for every Linux user regardless of how they installed the app. The complaint surface was the usual mix of "updater is broken on my distro-package install" issues with no actionable path because the gate had no platform awareness.

## What it changes

Refactors the gate into a pure function `isUpdaterEnabled({ isPackaged, channel, platform, appImage })` at the new file `packages/desktop-electron/src/main/updater-support.ts:7-13`, and rewrites `constants.ts:11-15` to call it with `{ isPackaged: app.isPackaged, channel: CHANNEL, platform: process.platform, appImage: process.env.APPIMAGE }`. The function body is four lines: bail false if not packaged, bail false on dev channel, return true if not Linux, and on Linux return `Boolean(input.appImage)`. The accompanying test file at `updater-support.test.ts:1-55` covers all four branches: unpackaged → false, dev channel → false, packaged win32/prod → true, packaged linux/prod with `appImage: undefined` → false, packaged linux/prod with `appImage: "/tmp/OpenCode.AppImage"` → true.

## Design analysis

The `APPIMAGE` env-var sniff is the correct primitive for this detection — AppImage's runtime stub sets `APPIMAGE=/path/to/the.AppImage` for the spawned process before exec'ing the application, and every other Linux packaging format leaves it unset. Using it instead of a more elaborate "look at `process.execPath` and check for AppImage signatures" check trades zero correctness (false positives would require a user to manually export `APPIMAGE` to a non-AppImage path, which is its own user error) for a one-line check. The function-extraction shape is exactly right: the previous one-line ternary at `constants.ts:10` had no test surface at all because it was evaluated at module import against `app.isPackaged` (a process-wide singleton you can't easily mock in bun:test); pulling the logic into a pure 4-arg function makes all four branches trivially testable without an Electron stub. The test trio at `updater-support.test.ts:32-51` covers the platform = `darwin` (true via the `!== "linux"` short-circuit), `win32` (true via the same path), and both Linux branches — that's correct minimum coverage for the four non-trivial branches.

## Risks

Very low. The single behavior change is "Linux non-AppImage installs no longer attempt auto-update" which is a strict reduction of the broken surface area — the previous code's failure mode on those installs was a confusing dialog followed by no actual update. Three latent considerations: (1) `process.env.APPIMAGE` is read once at module load via `constants.ts:14`, which matches the previous module-load-time behavior of `app.isPackaged && CHANNEL !== "dev"` — no subtle re-evaluation difference. (2) The test imports from `./updater-support` with the bun:test runner; the test file lives in the same package as the source, so the test runner doesn't need to know about Electron at all (which would have been a problem since `electron` is hard to load outside an Electron process). (3) Snap's confinement may prevent `APPIMAGE` from being inherited from a parent shell, but Snap installs of opencode-desktop are not the AppImage format anyway — they correctly return false on this gate. No regression risk.

## Suggestions

Minor and non-blocking: the `appImage: undefined` case at `:69-71` and `:74-79` could share a single `test.each` to halve the file length, but the explicit "describe(linux)" form is arguably more readable for documentation purposes. The `UpdaterSupportInput` type at `updater-support.ts:1-6` could narrow `platform` to the union of `"darwin" | "linux" | "win32"` if the project wants to enforce that callers don't pass exotic platforms (e.g. `freebsd` would currently return `true` from this function on packaged prod builds, which is technically wrong but matches the previous behavior). Neither is blocking — both are stylistic improvements that can land in a follow-up when someone touches this module again.

## What I learned

`APPIMAGE` env-var presence is the canonical AppImage-runtime detection signal — set by the AppImage runtime stub before exec'ing the embedded binary, unset everywhere else. Cleaner than `process.execPath.endsWith(".AppImage")` (which doesn't work — execPath points to the extracted binary inside the squashfs mount, not the AppImage file itself) and cleaner than checking `/proc/self/maps` for the squashfs mount. Worth remembering the next time any cross-platform Electron app needs to gate behavior on Linux packaging format.
