# QwenLM/qwen-code #3627 — feat: add macOS desktop app installer

- **Head SHA:** `7df9f2970b4ea7aaba733759eae370f88a4a19d4`
- **Base:** `main`
- **Author:** huangrichao2020
- **Size:** +387 / −0 across 5 files (README link, docs page, install script, icon binary, docs nav)
- **Verdict:** `merge-after-nits`

## Summary

Optional macOS launcher: `bash scripts/installation/install-qwen-macos-app.sh`
generates a `Qwen Code.app` bundle (an AppleScript applet that does
`tell application "Terminal" … do script "qwen"`) and installs it to
`/Applications` (or falls back to `~/Applications` when the system Applications
dir isn't writable). Adds a docs page at
`docs/users/installation/macos-desktop-app.md` and a README link with the
one-clone and one-curl install paths. Replaces closed PR #3564 with carried-over
review fixes around `BASH_SOURCE` safety, system/user install fallback policy,
and bundled-icon handling.

## What's right

- **Bash is re-`exec`ed safely if the script is invoked under `sh`.** The
  prologue at `install-qwen-macos-app.sh:13-19` checks `${BASH_VERSION}` and
  `exec bash -- "${0}" "$@"` if it's not set. This handles the
  `bash -c "$(curl … | sh)"` invocation path that the no-clone install in
  the README uses. Necessary because the script later relies on bash arrays
  and `[[` syntax.

- **`set -euo pipefail` plus `trap 'rm -rf "${TMP_DIR}"' EXIT`** at lines
  21-22 and 145. Strict mode plus deterministic temp cleanup is the right
  posture for a curl-pipe-bash installer.

- **The "system app exists but `/Applications` not writable" stop-and-error
  branch at lines 124-135 is the correct policy.** Without this, the
  installer would silently install a second copy under `~/Applications`,
  Spotlight/Launchpad would happily index both, and a user fixing a stale
  install would be debugging "I removed the app but Cmd+Space still opens
  the old one." The error message tells the user exactly what to do
  (`sudo rm -rf '${SYSTEM_APP_PATH}'`).

- **`BASH_SOURCE[0]:-` guard at line 51** uses the parameter expansion
  default to avoid the unbound-variable failure when the script is
  invoked as `bash -c "$(curl …)"` (no source path). The downstream
  `LOCAL_ICON_PATH` check at line 53 then degrades cleanly to the
  download path when no source dir is known. This is the carried-over
  review fix from #3564 and it's correct.

- **Icon pipeline is conventional.** `sips` to generate the `iconset` at
  9 standard sizes (16, 16@2x, 32, 32@2x, 128, 128@2x, 256, 256@2x, 512,
  512@2x), `iconutil -c icns` to compile, copy to
  `Contents/Resources/applet.icns` (the AppleScript applet's default icon
  slot). Well-formed.

- **`qwen --version` precondition check at lines 89-97** is the right
  precondition (the .app does nothing if `qwen` is not on PATH at app
  launch). The error message points at the official install URL.

- **Idempotent rebuild path: `rm -rf "${APP_PATH}"` then `cp -R …` at
  lines 192-193.** Reinstall on top of an existing app is clean — no
  partial-update risk.

- **Docs are accurate to the script.** The `Reinstall stops because an
  existing /Applications app is not writable` section at
  `macos-desktop-app.md:103-114` documents exactly the branch at
  `install-qwen-macos-app.sh:124-135`. The `--auto` flag mentioned in the
  script header at line 5 is implemented at lines 33-37.

## Concerns / nits

1. **`INSTALL_URL` is hard-coded to a `qwen-code-assets.oss-cn-hangzhou.aliyuncs.com`
   bucket** at line 53. If that bucket ever rotates or the project moves
   off Aliyun OSS, every cached copy of this script in user shells will
   keep pointing at a stale URL. Two safer alternatives: (a) point at the
   GitHub-hosted `https://raw.githubusercontent.com/QwenLM/qwen-code/main/scripts/installation/install-qwen.sh`
   the way `ICON_URL` at line 54 already does (consistent with itself,
   single trust domain), or (b) read it from a tagged release artifact
   that has a stable redirect.

2. **No `curl` precondition check for the no-clone install path.** The
   no-clone path needs `curl` for both the script-fetch (run by the user's
   shell, not by the script) and for the icon download at line 158. The
   script tests `command -v curl >/dev/null 2>&1 && curl …` so the icon
   path degrades gracefully, but a system without curl on the no-clone
   path can't even reach line 1 of the script. Worth a one-line
   `command -v curl >/dev/null 2>&1 || { echo "curl required"; exit 1; }`
   near the macOS check at line 79.

3. **`do script "qwen"` at the AppleScript level is shell-injectable if
   the binary name is ever templated.** Today it's a literal `"qwen"` so
   there's no injection surface, but the `Customization` section in the
   docs at `macos-desktop-app.md:138-145` shows users editing the script
   to add args (`do script "qwen --model qwen-plus"`). That's fine for a
   user-typed value, but if the install script ever grows a
   `--launch-args="…"` flag, the args would need shell-escaping before
   substitution into the AppleScript template. Not a current bug, just
   future-proofing.

4. **No Gatekeeper / quarantine mention in docs.** `osacompile`-built
   `.app` bundles installed via `cp -R` from a downloaded shell script
   *will* trigger Gatekeeper's "App is from an unidentified developer"
   prompt on first launch (and on subsequent launches if the user has
   strict Gatekeeper). The docs should call out the user's option to
   right-click → Open the first time, or to `xattr -d
   com.apple.quarantine "/Applications/Qwen Code.app"`. Without this,
   users will hit the warning and assume the installer is broken.

5. **Reinstall doesn't preserve user customization.** If a user edits the
   AppleScript per the `Customization` section of the docs, then reruns
   the installer, their edits are clobbered (the `rm -rf "${APP_PATH}"`
   at line 192 is unconditional). At minimum the docs should warn users
   that customization persists only across script runs if they save their
   edits separately. Better fix: detect a `Resources/.user-customized`
   marker file and skip the rebuild if present.

6. **`uname` check at line 81 only blocks non-Darwin** but not
   architecture or macOS version. The docs at `macos-desktop-app.md:13`
   say "macOS 12 or later" but the script doesn't enforce it. `osacompile`
   has been stable across macOS versions so this is probably moot, but a
   `sw_vers -productVersion` check would let the script fail with a clear
   message rather than producing an app that won't launch.

7. **The icon file `scripts/installation/qwen-icon.png` is in the repo as
   a binary blob with `additions: 0, deletions: 0`** (the diff doesn't
   show byte content). Two ops concerns: (a) PNG diffs don't review well
   in code-review tools, so a future icon update is invisible; (b) if the
   icon ever rotates, every clone gets the binary delta. Lower-impact
   alternative: store the icon as a base64 blob in the script and decode
   at install time, OR pull it from the same GitHub raw URL as the
   no-clone path uses. The bundled-clone path at lines 50-58 does
   prefer the local file when available, so if the binary stays small
   this is fine — just worth a comment in `CONTRIBUTING.md` about how to
   regenerate it.

8. **Minor:** `install-qwen-macos-app.sh:133` says "Or reinstall from an
   account that can write to ${SYSTEM_APP_DIR}" — but on macOS the
   typical fix is `sudo bash install-qwen-macos-app.sh`, not switching
   accounts. A more direct hint would help: "Or rerun this installer
   under sudo to write to /Applications."

## Verdict rationale

Optional convenience installer with no impact on existing users (the
`brew`/`npm` paths are untouched, this is purely additive at
`README.md:80-93`). The script is well-defended (BASH_SOURCE guard,
strict mode, atomic rebuild, system/user fallback policy with the
correct stop-and-error branch). Concerns are docs polish (Gatekeeper),
trust-surface cleanup (Aliyun-vs-GitHub URL), and minor robustness gaps —
none block merge.
