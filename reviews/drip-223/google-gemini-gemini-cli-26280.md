# google-gemini/gemini-cli#26280 — fix(build): detect Bun runtime in build scripts to avoid hardcoded npm

- Repo: google-gemini/gemini-cli
- PR: https://github.com/google-gemini/gemini-cli/pull/26280
- Head SHA: `c9f63c794087`
- Size: +44 / -8 (three files)
- Verdict: **merge-after-nits**

## What the diff actually does

Three coordinated changes so a Bun-only contributor isn't blocked at
two boundaries (the `prepare` script chain that triggers
`scripts/build.js`, and the husky `pre-commit` hook).

1. **`.husky/pre-commit:1–3`** — prepends a one-line shell pm-detect:
   ```sh
   if command -v npm >/dev/null 2>&1; then PM=npm; else PM=bun; fi
   ```
   then changes the body from `npm run pre-commit ||` to `"$PM" run
   pre-commit ||`. The `command -v` test is portable POSIX. Note the
   ordering bias: `npm` is preferred when both are available, which
   matches "npm users are the default, Bun users are the fallback."

2. **`scripts/build.js:24–32`** — adds an `isBun` boolean computed as:
   ```js
   const isBun =
     !!process.versions.bun ||
     !!process.env.npm_config_user_agent?.includes('bun');
   ```
   The two-arm check is load-bearing: `process.versions.bun` is true
   only when the *current* JS process is Bun, but `bun run build` shells
   out to `node scripts/build.js` (per package.json), so the inner Node
   process needs the env-var arm. The comment at `:27–30` names this
   rationale explicitly, which is the right "future-bisecter
   discoverability" shape.

3. **`scripts/build.js:34–63`** — three `execSync` call sites flip from
   unconditional `npm` to conditional:
   - `npm install` → `bun install` (line 36)
   - `npm run generate` → `bun run generate` (line 41)
   - The CI/non-CI workspace-build branch is replaced by a 3-way
     split: `isBun` ⇒ explicit core-first, then parallel-rest;
     `process.env.CI` ⇒ existing sequential `npm run build --workspaces`;
     else ⇒ existing concurrently-driven path.
   The Bun arm at `:48–58` builds `@google/gemini-cli-core` *first*
   (`bun run --filter @google/gemini-cli-core build`) and then
   *everything except core* in parallel
   (`bun run --filter '*' --filter '!@google/gemini-cli-core' --parallel build`),
   with a comment naming why: "bun run --filter does not respect
   topological order, so build core explicitly first (everyone depends
   on it) before the rest." This is a real Bun gotcha — it doesn't
   honor `dependencies` for filtered builds the way `npm`'s
   `--workspaces` does, so manual ordering is the only option.

4. **`scripts/build_package.js:35–50`** — duplicates the `isBun` detect
   (with a `// See scripts/build.js for why both checks are needed.`
   pointer-comment) and conditionally swaps `bundle:browser-mcp`'s
   driver between `npm` and `bun`.

## Why merge-after-nits

The fix is correct, surgical, and addresses a real "bun-only contributor
can't commit" workflow break. The nits are cleanup-grade.

## Nits

1. **Duplicated `isBun` detect** at `build.js:28–30` and
   `build_package.js:38–40` is exactly the kind of two-site
   redefinition that drifts in 6 months when one site adds a new
   detection arm (e.g., `process.env.BUN_INSTALL`) and the other
   doesn't. A 5-line `scripts/lib/runtime.js` exporting `isBun` would
   make the two sites byte-identical imports. The pointer-comment
   acknowledges the duplication but doesn't fix it.
2. **The husky preference order in `.husky/pre-commit:1`** prefers
   `npm` over `bun` when both are present. This matches "npm is
   default" but means a contributor who *wants* to commit under bun
   (because their toolchain is bun-only and they have npm installed
   for some other reason) silently gets npm. A `BUN_PREFERRED=1`
   env-var override (or honoring `npm_config_user_agent` here too)
   would make the preference flippable without editing the hook. Not
   a blocker — the current behavior is correct for the documented
   "npm-default, bun-fallback" intent.
3. **No test or CI lane covers the Bun build path.** The PR description
   doesn't claim CI runs under Bun, so the new code paths
   (`bun install`, `bun run --filter '*' --filter '!@core' --parallel`)
   are exercised only by the contributor's local machine and any
   downstream Bun users. A minimal "smoke" job in `.github/workflows/`
   that runs `bun install && bun run build` against a tiny matrix
   entry would catch silent regressions when the upstream
   `bun run --filter` semantics change.
4. **`process.env.npm_config_user_agent?.includes('bun')`** is the
   right detection but technically matches anything containing `bun`
   in the user agent, including hypothetical future tools whose UA
   string mentions bun for compat. Anchoring to the `^bun/` prefix
   (which is bun's actual UA shape: `bun/1.x.y npm/?`) would be
   tighter, but the false-positive risk of the looser match is
   essentially nil today.

## Theme tie-in

Fix the runtime-mismatch at the *script-driver boundary* (one
`isBun` constant feeding the `execSync` arm) rather than at every
package's `prepare` script. The Bun-specific topological-order
workaround is the kind of contract drift that needs the explicit
comment the PR provides — without it, a future maintainer who reads
"why does this PR build core separately" would have no idea.
