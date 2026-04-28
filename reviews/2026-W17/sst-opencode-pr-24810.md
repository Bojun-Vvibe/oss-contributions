# sst/opencode PR #24810 — upgrade opentui to 0.2.0

- URL: https://github.com/sst/opencode/pull/24810
- Head SHA: 8682a32922b5
- Files: `bun.lock`, `package.json`, `packages/plugin/package.json`
- Verdict: **merge-after-nits**

## Context

Bumps `@opentui/core` and `@opentui/solid` from `0.1.105` to
`0.2.0` across the root `package.json`, the plugin package's
`peerDependencies` (both `>=0.1.105` → `>=0.2.0`), and the
corresponding lockfile entries. Pure dependency upgrade with no
source changes.

## Design analysis

The `package.json` and `packages/plugin/package.json` edits are
exactly what you'd expect: pinned exact version at the root,
relaxed `>=` peer constraint in the plugin package. The lockfile
diff is large and mostly noise from the optional platform-arch
sub-packages; the substantive bits are:

1. New transitive dependencies pulled in by `@opentui/core@0.2.0`:
   `string-width@7.2.0` and `strip-ansi@7.1.2`. These are
   well-trodden libraries, no concern.
2. `bun-webgpu` jumps from `0.1.5` → `0.1.7` (the
   `optionalDependencies` chain).
3. `babel-preset-solid` bumps `1.9.10` → `1.9.12` and the
   `solid-js` peer constraint moves `1.9.11` → `1.9.12`. Worth
   double-checking the project's pinned `solid-js` version is
   compatible, otherwise the peer warning could mask a real
   incompatibility.
4. A `babel-preset-solid@1.9.10` entry remains under the
   `opentui-spinner/@opentui/solid/` lockfile path, suggesting
   the `opentui-spinner` workspace is still pinned to the older
   `@opentui/core@0.1.105` — i.e., this is a partial migration.
   That's fine if intentional (spinner is a separate package
   that doesn't need to track the main pin) but worth confirming.

## Risks / suggestions

1. **No upgrade notes in PR description.** A 0.1.x → 0.2.0
   version bump on a TUI primitive almost certainly has breaking
   API changes. Even if opencode's usage doesn't trip them, a
   one-paragraph "what we tested" (snapshots regenerated? smoke
   test of N TUI panels?) would dramatically de-risk the merge.
2. The `opentui-spinner` workspace staying at `0.1.105` means
   two copies of `@opentui/core` are now resolved at runtime —
   bigger bundle, possibility of two singletons. Acceptable, but
   add a tracking issue to bring spinner up too.
3. Recommend re-running the TUI snapshot tests in CI as a
   pre-merge gate; visual regressions in TUI are easy to miss in
   manual review of a deps-only diff.

## What I learned

Major-component-version dependency PRs that touch only the
lockfile are almost never as safe as their diff suggests. The
real review surface is the upstream changelog, not the diff.
When the PR description says only "upgrade opentui to 0.2.0",
the reviewer's first move should be to open the upstream
0.2.0 release notes — and the PR author should have linked them.
