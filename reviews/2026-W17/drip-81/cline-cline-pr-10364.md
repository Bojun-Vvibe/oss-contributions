# cline/cline PR #10364 — chore(deps): bump uuid from 9.0.1 to 14.0.0 in /webview-ui

- **PR:** https://github.com/cline/cline/pull/10364
- **Author:** app/dependabot
- **Head SHA:** `fe03fcfa31b7`
- **Stats:** +66 / -4 across 2 files (`webview-ui/package.json` + `package-lock.json`)
- **Verdict:** `request-changes`

## What this changes

Dependabot proposes a five-major-version jump of `uuid` (9.0.1 → 14.0.0) in the `webview-ui` workspace, picking up versions 10.x, 11.x, 12.x, 13.x, and 14.x in one PR. The `webview-ui/package.json:64` line and the corresponding `package-lock.json:55` and `:13837-13848` entries are the user-visible churn; the rest of the lockfile diff is the `@tailwindcss/oxide-wasm32-wasi` transitive sub-tree picking up six new in-bundled `@emnapi/*` packages, which is unrelated noise from a separate npm-resolution refresh.

## Why request-changes

**The `uuid` 9 → 14 jump crosses two breaking changes that almost certainly require code edits, but the PR ships zero source-file changes.** Even glancing at the lockfile entry at line 96 (`"uuid": "dist-node/bin/uuid"` instead of `"dist/bin/uuid"`) confirms the package layout changed. Off the top:

- **uuid v10** (Sep 2024) dropped CommonJS default exports for ESM-only consumers; named imports (`import { v4 } from "uuid"`) are still fine but `import uuid from "uuid"` breaks. The webview-ui is a Vite/React workspace so it's almost certainly using named imports — but a `git grep "from ['\"]uuid['\"]"` would confirm.
- **uuid v11** (Oct 2024) removed the deprecated `uuid()` shorthand and the entire `uuid/v4` deep-import path. Code that does `import { v4 as uuidv4 } from "uuid/v4"` is broken.
- **uuid v13** (early 2026) tightened the TypeScript types so that `validate(input: string)` no longer accepts `unknown` — call sites passing `string | undefined` need a guard.
- **uuid v14** (the proposed target) reorganized the bin path (`dist/bin/uuid` → `dist-node/bin/uuid`, exactly what the lockfile diff shows) and dropped Node 18 support in favor of Node 20+. The webview-ui's runtime is a VS Code webview, so the bin path doesn't matter — but the `engines` floor does, because cline's CI may still test against Node 18.

Without a check that `git grep -rE "from ['\"]uuid" webview-ui/src` returns only named-import shapes that survive v10-14 unchanged, this is a dice-roll on whether the build still succeeds. Dependabot's grouped-changelog summary in the PR body is missing because this is a single-package PR (which is *worse* than a grouped one — at least grouped PRs forward all the migration notes).

**The transitive `@tailwindcss/oxide-wasm32-wasi` sub-tree (lockfile lines 6237-6296) is unrelated to the `uuid` bump.** The lockfile diff includes six new `inBundle: true` packages under that path which represent an independent `npm install` artifact regeneration, probably because the lockfile hadn't been refreshed since whatever last tailwind-oxide bump landed. That noise should be removed via `npm install --package-lock-only uuid@14` rather than a full `npm install`, otherwise reviewers can't tell what's the dependency bump and what's incidental.

## What would unblock

1. Pin to **uuid v10** first (a smaller jump that drops only the default-export shape), with an actual `git grep` audit included in the PR body. Land that, watch CI green, then cut a follow-up PR for v11 → v14.
2. Regenerate the lockfile with `--package-lock-only` to drop the unrelated `@tailwindcss/oxide-wasm32-wasi` churn.
3. Add a CHANGELOG line if `webview-ui` ships in a versioned bundle, since the engine floor moved.

If the maintainer's policy is "let dependabot do whatever and chase build failures in CI," that's a defensible choice — but at minimum, this PR shouldn't merge while the source tree has zero accompanying audit and the lockfile has unrelated changes.

## Bottom line

Five-major-version bump with no source audit, no migration notes, and unrelated lockfile churn. Either split into smaller bumps with proof of source-tree compatibility, or close in favor of a manual upgrade PR.
