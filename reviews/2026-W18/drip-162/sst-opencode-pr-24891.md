# sst/opencode PR #24891 — feat(tool): add open tool for files and URLs

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/24891
- Head SHA: `1ed488afc9`
- State: OPEN, +417/-0 across 6 files (closes #24890)

## What it does

Adds a new built-in `open` tool that surfaces a local file, folder, `file://` URL, or `http(s)` URL via the OS default handler (`open` on macOS, `xdg-open` on Linux, `rundll32 url.dll,FileProtocolHandler` / `explorer.exe` on Windows). Wires the tool into the `AVAILABLE_PERMISSIONS` allowlist (`agent.ts:34`), the TUI inline render arm (`session/index.tsx:1596-1598`, with the `Open` component at `:2233-2253`), and the registry. Schema rejects URL schemes outside `{http, https, file}`, gates execution behind `ctx.ask({permission: "open", ...})`, and records `delivered` based on launcher exit code with a Windows special-case (`explorer.exe` non-zero counts as delivered).

## Specific reads

- `tool/open.ts:14-37` — `Parameters` schema. `target` is `MinLength 1, MaxLength 2048`. Optional `reason` (≤400 chars) and `reveal_in_folder` boolean. No `cwd`/`directory` override — resolution is implicitly project-rooted via `ins.directory`. Good: model can't escape the project root by passing a `directory:` arg. Implicit risk: it can still pass `target: "../../../../../etc/passwd"` and the tool will resolve+stat+open it (see below).
- `tool/open.ts:50-67` — `classifyTarget` uses scheme regex `^([a-z][a-z0-9+.-]*):/i` and explicitly rejects single-letter schemes to avoid mis-identifying `C:\foo` as a URL. Sound, but `file:C:/foo` (no slashes) parses through `new URL` and lands in the URL branch — and then `fileURLToPath` converts it. Worth a unit test pinning that branch on Windows.
- `tool/open.ts:138-150` — `platformLauncher` for Linux always uses `xdg-open`. No fallback to `gio open` / `kde-open5` / `wslview`. WSL users in particular hit a sharp edge here (Linux platform, but `xdg-open` may not be present and won't traverse to Windows). A short comment + `try xdg-open || try wslview || try gio open` chain would broaden coverage; otherwise just document the single-binary contract.
- `tool/open.ts:212-218` — explicit `ALLOWED_SCHEMES = {http, https, file}` enforcement. Correct.
- `tool/open.ts:236-241` — relative path branch: `path.resolve(ins.directory, classified.raw)`. **Does not check the resolved path is under `ins.directory`** — `target: "../../../../../etc/shadow"` resolves outside the project root and gets `stat`'d + `open`'d. The permission gate (`ctx.ask`) limits the blast radius to "user must approve once with `always: ["*"]`", but the always-pattern is `"*"` not `${ins.directory}/**`, so a single approval grants the model unlimited filesystem read via the OS default handler. Recommend either (a) constraining `always` to the resolved-target's containing dir, or (b) refusing resolution that escapes `ins.directory` unless an explicit `allow_outside_project: true` is passed.
- `tool/open.ts:243-256` — `ctx.ask` with `permission: "open"`, `patterns: [permissionTarget]`, `always: ["*"]`. The `always: ["*"]` is the crux of the concern above. For a `file:///etc/passwd` URL the `permissionTarget` correctly is the resolved path, but the `always` allowlist is still `*`.
- `tool/open.ts:280-283` — Windows `explorer.exe` exit-code special case: `delivered = code === 0 || (win32 && cmd === "explorer.exe" && code !== null)`. Documented in comment ("can exit non-zero even when it successfully hands work to an existing Explorer process") and matches what tools like VS Code do. Worth checking: `code !== null` includes the timeout case (`runProcess` resolves `code: null, stderr: "timeout"`), so a hung explorer is correctly reported as not-delivered. Good.
- `tool/open.ts:118-126` — `runProcess` 8-second timeout with `child.kill()` on expiry, plus `signal.addEventListener("abort", ...)`. The `child.kill()` call has no fallback `kill('SIGKILL')` after a grace window, so a misbehaving `xdg-open` that ignores SIGTERM hangs the tool until the surrounding ctx.abort fires.
- TUI `Open` component (`session/index.tsx:2233-2253`): label truncation at 60 chars for URLs (`t.slice(0, 57) + "..."`) but uses `path.basename(t)` for file/dir kinds — file basenames over 60 chars (e.g. data files) won't truncate. Minor.

## Risk

1. **Path-escape via relative target + permanent-grant** (highest). A `target: "../sibling-project/.env"` resolves outside `ins.directory`, surfaces `ctx.ask` once, and once the user clicks "always" with the `"*"` pattern, every future `open` call (including `target: "/etc/shadow"`) skips the prompt. The permission key should be tighter — at minimum, scope `always` to a sibling-of-or-under `ins.directory` glob, or split into two perms (`open:project` vs `open:anywhere`).
2. **No anti-traversal stat**: even without "always", a single one-shot approval clicks through to opening files outside the workspace. For a security-positioned product this is worth a pre-stat path-canonicalization-vs-`ins.directory` check by default with an explicit override.
3. **Linux fallback**: `xdg-open`-only is brittle in containers and minimal images. A two-line `try-fallback` chain or an explicit "requires xdg-utils" doc note would reduce confused-user issues.
4. Test surface (`test/tool/open.test.ts` — referenced in file list, +N LOC) — couldn't read here, but should pin: (a) URL scheme rejection, (b) relative-path resolution, (c) `file://` URL → path coercion, (d) Windows reveal-in-folder, (e) timeout fallthrough, (f) escape-from-project rejection (missing today).

## Verdict

`merge-after-nits` — feature is well-shaped, the schema/permission/launcher decomposition is right, and the `delivered` heuristic for `explorer.exe` shows real-world testing. The two structural nits worth fixing pre-merge are (1) the `always: ["*"]` permission grant (too broad given OS-handler can read anything), and (2) no canonicalize-vs-project-root check on the relative-path branch. Both are 5-line fixes. Linux fallback and test coverage are merge-after-merge nits.

## What I learned

OS-default-opener tools sit in an awkward security pocket: they're not file-read tools (the agent doesn't see the bytes) but they *do* exfiltrate state to the user's GUI, and on the URL side they can trigger arbitrary browser navigations (telemetry pings, `oauth://...` callbacks if the user-agent allows non-`{http,https,file}` schemes — though this PR closes that with the allowlist). The right permission-grant shape is "scope-bounded with explicit override" not "blanket `*`".
