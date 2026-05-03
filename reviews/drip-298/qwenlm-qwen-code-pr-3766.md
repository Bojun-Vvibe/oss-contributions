# Review: QwenLM/qwen-code PR #3766 — chore(release): v0.15.6

- **Head SHA:** `cf5f447fdd598f995a2d9392c9254cca46b7461f`
- **State:** MERGED (2026-04-30)
- **Size:** +25 / -25, 12 files
- **Verdict:** `merge-as-is`

## Summary
Automated release PR that bumps every `package.json` (and the lockfile) from
`0.15.3` → `0.15.6`, and the sandbox image URI in the root `package.json`
from `ghcr.io/qwenlm/qwen-code:0.15.3` → `:0.15.6`.

## Diff inventory
- `package.json:3` and `package-lock.json:3` — root version bump.
- `package.json:21` — `config.sandboxImageUri` retagged to `:0.15.6`.
- 10 workspace `package.json`s under `packages/channels/{base,dingtalk,plugin-example,telegram,weixin}/`,
  `packages/{cli,core,vscode-ide-companion,web-templates,webui}/` — each gets
  `"version": "0.15.6"` in place of `"0.15.3"`.
- `package-lock.json` — symmetric bump for every workspace package node
  (`packages/channels/base:16863`, `packages/cli:16921`, `packages/core:17580`,
  `packages/vscode-ide-companion:21024`, `packages/web-templates:21271`,
  `packages/webui:21799`).

## What's correct
- All workspace versions march forward together — no drift between `cli` and
  `core` peer-dep ranges.
- `package-lock.json` updates mirror every workspace bump 1:1, no orphan
  `0.15.3` references left behind.
- Sandbox image URI bump is colocated with the version bump, so a tag-based
  release won't push a binary that pulls a stale image.

## Why not "merge-after-nits"
Because there is nothing to nit — this is a mechanically generated release
PR with no behavioral surface, no new code paths, and no human-authored
prose to second-guess. The release pipeline that generated it owns
correctness; reviewing the diff line-by-line is the same as reviewing the
output of `npm version`.

## What I'd want to see in the *next* release-bot tick (not blocking)
- A CHANGELOG link in the PR body. Right now the body is a single line
  ("Automated release PR for v0.15.6. Syncs package.json versions on main.")
  which doesn't tell a reviewer or downstream consumer what changed.
- An explicit confirmation in CI that the version jumped by exactly one minor
  step expected by the release tag (here `0.15.3 → 0.15.6` skips two patch
  versions — presumably 0.15.4 and 0.15.5 were never published, but a
  reviewer can't tell without external context).
- A guard against the sandbox image being pushed to `ghcr.io` *after* the
  npm publish, so users on `0.15.6` never momentarily resolve a missing image.

## Verdict rationale
Mechanical, internally consistent, already merged. No code changes, no
security surface, no API change. Approve.
