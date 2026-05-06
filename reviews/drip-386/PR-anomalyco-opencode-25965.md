# anomalyco/opencode#25965 — docs: update desktop app references from Tauri to Electron

- **Head SHA**: `73d2526d34f912f3893d690a4d97a639f0b8d15e`
- **Stats**: +55 / -67, ~25 files (CONTRIBUTING.md + every localized README.\*.md)

## Summary

Pure docs sync after the desktop packager flipped from Tauri to Electron. Two parallel mechanical changes: (1) `CONTRIBUTING.md` rewrites the "Running the Desktop App" section to describe the Electron dev/build/package commands instead of `bun run --cwd packages/desktop tauri dev|build`; (2) every translated README updates the download-asset table from `opencode-desktop-darwin-aarch64.dmg` / `darwin-x64.dmg` to `opencode-desktop-mac-arm64.dmg` / `mac-x64.dmg` to match what `electron-builder` actually emits (electron-builder's default macOS arch tokens are `arm64` and `x64`, not the `darwin-*` strings Tauri used).

## Specific citations

- `CONTRIBUTING.md:73`: `packages/desktop` description flips "built with Tauri" → "built with Electron". Subsequent block at lines 123–143 drops the `tauri dev` / `tauri build` invocations and the Tauri-prerequisites NOTE, and replaces them with `bun run --cwd packages/desktop dev` and a `build` + `package` two-step. The deletion of the prerequisites NOTE is *correct* — Electron doesn't need the Rust toolchain — but the new section silently loses the explicit "this opens the native window" sentence; a fresh contributor running `bun run --cwd packages/desktop dev` no longer gets a hint about what to expect. Minor doc nit, not a blocker.
- `README.md:68-74`: the canonical English table updates filenames AND tightens column widths (the header separator shrinks from 39 dashes to 36 to match the shorter content). The translated tables at e.g. `README.bn.md:73`, `README.de.md:73`, `README.es.md:73`, `README.fr.md:73`, `README.ja.md:73`, `README.ko.md:73`, `README.zh.md:73`, `README.zht.md:73` (20 locale files in total) get the *content* swap but **not** the column-width retightening — so most localized tables now have a stale `------------------------------------- ` separator under a shorter `mac-arm64.dmg` cell. Renders fine in markdown viewers but is visually misaligned in monospace/diff tools. Either tighten all separators or leave English alone for consistency.
- `README.md:74` and `README.bn.md:73` switch the Linux row from `AppImage` to `` `.AppImage` `` (backticked, with leading dot) — a small but inconsistent improvement that wasn't applied to the other ~18 locale files. They still say bare `AppImage`. Pick one: either ship the backtick treatment everywhere or revert it in `.md` and `.bn.md`.
- No code is touched (no `packages/desktop/**` source changes); this PR is documentation-only and can't break a build. The matching code change presumably landed in the package.json/build.yml in an earlier PR; verify by grepping for `tauri` under `packages/desktop/**` before merge — any leftover scripts there would make this docs-PR ship a lie.

## Verdict

**merge-after-nits**

## Rationale

Mechanical, low-risk, and obviously correct in spirit — the prior docs were stale relative to the build system. Two rough edges: (a) the table column-separator width is updated only in English `README.md`, leaving 18+ localized tables with mismatched separators; (b) the Linux-asset backtick treatment is applied inconsistently (only English + Bengali). Both are 30-second `sed` fixes the author can squash before merge. One process nit: a follow-up sweep for the literal token `tauri` under `packages/desktop/`, `package.json`, `script/`, and `.github/workflows/` would protect against the docs claiming Electron while a stray `tauri build` line still exists in CI — but that's outside this PR's scope. Nothing in the diff blocks merge once the cosmetic table alignment is squared.
