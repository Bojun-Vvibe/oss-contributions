# Review: google-gemini/gemini-cli #26280 — fix(build): detect Bun runtime in build scripts to avoid hardcoded npm

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26280
- **Author**: euxaristia
- **Head SHA**: `c9f63c7940876289a8dcfe5c4ee562ec0b6a1f1d`
- **Base**: `main`
- **Files**: `.husky/pre-commit` (+3/-1), `scripts/build.js` (+31/-4),
  `scripts/build_package.js` (+10/-3)
- **Verdict**: **merge-after-nits**

## Reasoning

Closes #26279. The three target scripts (`scripts/build.js`,
`scripts/build_package.js`, `.husky/pre-commit`) hard-code `npm`
invocations, which break on Bun-only systems where `npm` isn't on
`PATH`. The PR adds runtime detection so Bun users get `bun install` /
`bun run …` while npm users get the byte-identical pre-fix path.

What the diff does well:

- **Two-way runtime detection at `build.js:30-32` and
  `build_package.js:39-42`**: `isBun = !!process.versions.bun ||
  !!process.env.npm_config_user_agent?.includes('bun')`. The first
  arm catches the direct `bun scripts/build.js` invocation, the second
  catches `bun run build` → which per `package.json` invokes
  `node scripts/build.js` (Node child process), where
  `process.versions.bun` is absent but Bun has set
  `npm_config_user_agent` and the env inherits. The PR comment
  explicitly explains why both arms are required, which is the right
  thing to do for a future reader.
- **npm path is character-identical to pre-fix.** The `else if
  (process.env.CI)` branch at `build.js:60` and the `else` branch
  preserve the existing CI-vs-non-CI workspace-build split, so npm
  users see zero behavioral change. This is the load-bearing safety
  property for not regressing the dominant install flow.
- **Bun's `--filter` topological-order gap is handled correctly** at
  `build.js:48-58`: builds `@google/gemini-cli-core` explicitly first,
  then runs `bun run --filter '*' --filter '!@google/gemini-cli-core'
  --parallel build` for the rest. This matches Bun's documented
  `--filter` semantics (no DAG awareness) and the PR comment calls
  out the workaround.
- **`.husky/pre-commit:1-3`**: `if command -v npm >/dev/null 2>&1; then
  PM=npm; else PM=bun; fi` is the right shell idiom. POSIX-portable,
  no exec, no shellisms.
- **Honest scoping**: PR body explicitly calls out that the root
  `package.json` `bundle` script still uses npm-style `-w` /
  `--workspace=` flags that Bun handles differently, and that
  `bun install` will still fail downstream of this fix at the bundle
  step. Tracking it separately is the right call — keeping this PR
  scoped to the script-level npm-hardcoding called out in the issue.

Nits worth addressing before merge:

- **Detection helper duplication**: the same five-line `isBun = …`
  expression appears verbatim in both `build.js:30-32` and
  `build_package.js:39-42`. Worth extracting to a `scripts/util/runtime.js`
  exporting `isBun()` so the next contributor doesn't have to keep
  the two sites in sync. Small, but mechanical drift on detection
  logic is a real risk.
- **`npm_config_user_agent` parse is substring-match**: `.includes('bun')`
  will false-positive on a hypothetical npm user-agent containing the
  string "bun" anywhere (e.g. a path component). Vanishingly unlikely,
  but a stricter check like `^bun/` (after the `agent.split(' ')[0]`
  parse) is more robust. Optional.
- **No CI matrix coverage for the Bun path.** The PR test plan
  acknowledges this — checked manually on Bun-only system, npm
  branches verified by reading the diff. A follow-up adding a Bun
  job to CI would lock the contract and prevent regression. Out of
  scope for this PR but worth filing.
- **Auto-close concern**: PR body notes `Fixes #26279` is present
  from minute zero specifically to avoid the issue-link bot
  auto-closing again like #22341 did. Good defensive context.

## Suggested follow-ups

- File the `package.json` bundle-step rewrite as a separate issue/PR
  so Bun support is genuinely end-to-end, not just script-level.
- Consider documenting Bun support tier (best-effort? supported?) in
  `CONTRIBUTING.md` so contributors know whether to expect Bun-side
  CI to gate their PRs.
- Extract the runtime detection helper as noted above; add a one-line
  comment on the `npm_config_user_agent` path pointing at the Bun docs
  source for the env-var name.
