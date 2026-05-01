# QwenLM/qwen-code #3688 — chore(release): sdk-typescript v0.1.7

- **PR**: https://github.com/QwenLM/qwen-code/pull/3688
- **Head SHA**: `3a0561f4f308`
- **Files reviewed**: `package-lock.json`, `packages/sdk-typescript/package.json`
- **Date**: 2026-05-01 (drip-229)

## Context

Automated release-bot PR bumping the `@qwen-code/sdk` package from
0.1.6 → 0.1.7. Two-file mechanical bump: the package's own
`package.json` `version` field plus the regenerated entry in the
monorepo root `package-lock.json` for the `packages/sdk-typescript`
workspace.

## Diff (2 files, +2 -2)

`packages/sdk-typescript/package.json:3`:

```diff
   "name": "@qwen-code/sdk",
-  "version": "0.1.6",
+  "version": "0.1.7",
```

`package-lock.json:18210` (workspace entry mirror):

```diff
     "packages/sdk-typescript": {
       "name": "@qwen-code/sdk",
-      "version": "0.1.6",
+      "version": "0.1.7",
```

## Observations

1. **Two-pin consistency.** The root `package-lock.json` workspace
   entry must agree with the workspace's own `package.json`; both
   read `0.1.7` after this PR. npm CI would otherwise fail with a
   `EOVERRIDE` / `EWORKSPACE` error on `npm ci --workspaces`. The
   bot got it right.

2. **No source change.** The diff is exclusively version-string
   flips. There is nothing else to review.

3. **Release-PR pattern is well-trodden.** The repo has a documented
   automated release workflow (see #3766 `chore(release): v0.15.6`
   for the equivalent monorepo-wide bump shape and #3764 for the
   merge-back PR added to that workflow recently). This PR is the
   per-package companion that ships the SDK independently of the
   main CLI.

## Nits

- **Underlying changeset is opaque from this diff alone.** What
  changed in `@qwen-code/sdk` between 0.1.6 and 0.1.7 is not
  recoverable from these two files. A `CHANGELOG.md` under
  `packages/sdk-typescript/` (or a release-notes block in the PR
  body beyond "Automated release PR for sdk-typescript v0.1.7")
  would let downstream consumers decide whether to bump.

- **No CI assertion that the two pins agree.** Same recurring shape
  as other release-bumps in this drip — a one-line CI guard
  diffing `packages/sdk-typescript/package.json` `version` against
  the `package-lock.json` workspace entry would catch the
  "human edited one but not the other" failure mode if the bot is
  ever circumvented.

## Verdict

`merge-after-nits` — purely mechanical bot PR with the two
canonical version-pin sites correctly updated. Nits are
out-of-band (changelog discipline, CI guard) and shouldn't block
the release train.
