# QwenLM/qwen-code#3708 — chore(release): bump version to 0.15.3

- **PR**: https://github.com/QwenLM/qwen-code/pull/3708
- **Author**: @yiliang114
- **Head SHA**: `9295cfa86c8ff894cb2ec83bd993de615185ba9f`
- **Base**: `main`
- **State**: OPEN
- **Scope**: +25 / -25 across 12 files (mechanical version bump)

## Summary

Mechanical version bump from `0.15.2` to `0.15.3` across the monorepo. The 25+/25- exact symmetric file/line ratio is the load-bearing signature of a clean automated bump — every `0.15.2` becomes `0.15.3` and nothing else moves. 12 files cover the root `package.json` + `package-lock.json` + 10 workspace `package.json` files (channels: base/dingtalk/plugin-example/telegram/weixin; cli; core; webui; web-templates; vscode-ide-companion).

## Diff anchors (sample, all are exact `0.15.2` → `0.15.3`)

- `package.json:1-3` (root) — `"version": "0.15.2"` → `"0.15.3"`
- `package-lock.json:3-5, 12-13` — root + the lockfile's nested `packages.""` block both bump in lockstep. Lockfile-only bumps without a matching root `package.json` change are the canonical broken-bump signature; this PR has both.
- `package-lock.json:16863, 16873, 16884, 16898, 16911, 16921, 16963, 17580, ...` — every `packages/*/package.json` workspace gets its in-lockfile version mirror bumped. Lockfile-side workspace mirror is the second canonical broken-bump risk; this PR catches it.
- `packages/channels/base/package.json`, `packages/channels/dingtalk/package.json`, `packages/channels/plugin-example/package.json`, `packages/channels/telegram/package.json`, `packages/channels/weixin/package.json`, `packages/cli/package.json`, `packages/core/package.json`, `packages/web-templates/package.json`, `packages/webui/package.json`, `packages/vscode-ide-companion/package.json` — each has its `"version": "0.15.2"` → `"0.15.3"` line bumped.

## What review value exists beyond verifying the bump

For mechanical version bumps, the review surface reduces to:

1. **Provenance** — author is `@yiliang114`, who appears in prior bump PRs in this repo (drip history shows version-bump PRs from this account). Not a fresh-account contributor.
2. **Scope** — only `package.json` / `package-lock.json` files; no source code, no docs, no CI workflow files. Confirmed via the file list above.
3. **Shape** — exact 25+/25- symmetry, every changed line is the same `0.15.2 → 0.15.3` substitution. No collateral metadata edits (no `description` / `dependencies` drift in any of the 12 files).
4. **Workspace consistency** — all 10 workspaces bump together. A bump that left one workspace at `0.15.2` would risk a peer-dependency mismatch at install time.
5. **Lockfile integrity** — `package-lock.json` updates the workspace mirrors to match. Manual `package.json` bumps without `npm install` regenerating the lockfile would leave the lockfile pinning the old version and surface as a CI install failure.

## What I'd push back on (nits)

1. **No CHANGELOG entry visible in this diff.** A 0.15.3 patch release should carry at least one bullet naming what the patch closes (#3699 ask_user_question, #3705 preserve preview version overrides, etc., from drips 138-139). If the changelog lives elsewhere (e.g., GitHub Releases generated from PR labels), the PR body should link it; otherwise the version bump is opaque to consumers.
2. **`vscode-ide-companion`** — vscode-XXX surface in this monorepo, bumps in lockstep with the rest.
3. **No release-notes / signing-flow changes visible** — confirm by running `git log v0.15.2..HEAD --oneline` against the upstream tag that the bump corresponds to a coherent set of merged commits, not a bump-without-content.
4. **`@qwen-code/channel-base` is a `file:../base` workspace dep** — bumps work because npm resolves workspace versions through the workspace protocol, but a sibling channel pinning the dep at `^0.15.2` (rather than `file:../base`) would silently keep using 0.15.2 until its own bump. The `file:../` shape across channels means this risk doesn't apply here; worth noting for future fixed-version pin migrations.

## Verdict

**merge-as-is** — clean automated bump with the canonical 25+/25- symmetric ratio across root + lockfile + 10 workspaces. The only review value beyond shape verification is provenance (known author) and scope (no collateral edits). Recommend author add a CHANGELOG bullet or release-notes link in the PR body for traceability, but that's a process nit, not a code blocker.

Repo coverage: QwenLM/qwen-code (release-cut hygiene).
