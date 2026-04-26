---
pr: 24531
repo: sst/opencode
sha: 69dda83c2bd62fcce3fbf1b33051b5a3fa11a579
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24531 — upgrade opentui to 0.1.104

- **Head SHA**: `69dda83c2bd62fcce3fbf1b33051b5a3fa11a579`
- **Size**: +16 / −113 (the `bun.lock`/`package.json` version bump is small; the negative side is a 96-line deletion of `packages/opencode/src/cli/cmd/tui/util/terminal.ts`)

## Summary
Bumps `@opentui/core` and `@opentui/solid` peer/runtime deps from `0.1.103` → `0.1.104` (and the `0.1.103` minimum in `peerDependencies`), and *deletes* the local `Terminal.colors()` helper that did OSC 4/10/11 color queries. The PR body cites "bug fixes around theme mode feedback loop and indexed color mode" as the upstream rationale.

## Specific findings
- `bun.lock:509-510, 688-689` and the seven `@opentui/core-{platform}-{arch}@0.1.104` entries — all tagged checksums updated consistently. No version skew.
- `package.json:36-37` — pinned exact versions in `overrides` (or `resolutions` block; can't see context here). Consistent with the rest of the file's pinning style.
- **Deleted file `packages/opencode/src/cli/cmd/tui/util/terminal.ts`** — this exported `Terminal.colors()`, an OSC-escape-based foreground/background/palette query. The PR body says nothing about why this is being removed. Two scenarios:
  1. opentui 0.1.104 now provides this functionality natively → fine, but verify by searching for the old call sites and confirming they migrate to a `@opentui/core` API in this same PR. Not visible in the diff.
  2. The functionality was unused → fine but worth a one-line PR note.
- `packages/opencode/src/cli/cmd/tui/util/index.ts` — removes the `export * as Terminal from "./terminal"` re-export. If anything else in the codebase still does `Terminal.colors()`, the build will break. **Reviewer should run `rg "Terminal\.colors\("` and `rg "from .*util.*Terminal"` against the head SHA to confirm zero remaining call sites.**
- The OSC 10/11 query path the deleted file implemented is the same one users rely on for proper light/dark detection in tmux-on-Apple-Terminal scenarios where `waitForThemeMode` returns `null`. Confirm 0.1.104 re-implements OSC 10/11 (the `theme mode feedback loop` fix in the changelog suggests yes, but worth a manual smoke test in tmux + Apple Terminal + light scheme).
- No test changes. Acceptable for a dep bump but the terminal-color query path has no integration coverage to fall back on.

## Risk
Medium. Two unrelated changes (version bump + helper deletion) are bundled. If anything outside `app.tsx`/the SolidJS provider stack consumes `Terminal.colors()`, this regresses theming for those users.

## Verdict
**merge-after-nits** — call out the helper deletion in the PR body, link the upstream opentui changelog entry that obsoletes it, and verify zero remaining `Terminal.colors()` call sites at the head SHA.
