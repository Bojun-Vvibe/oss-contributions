# google-gemini/gemini-cli PR #26280 — fix(build): detect Bun runtime in build scripts to avoid hardcoded npm

- Link: https://github.com/google-gemini/gemini-cli/pull/26280
- SHA: `c9f63c7940876289a8dcfe5c4ee562ec0b6a1f1d`
- Author: euxaristia
- Stats: +44 / −8, 3 files

## Summary

The repo's build scripts and the husky `pre-commit` hook hard-code `npm`, which breaks contributors who use `bun install` and have no `npm` on PATH (the `prepare` script chain runs `scripts/build.js`, and the pre-commit hook fires on every commit). This PR adds a runtime detection — checking `process.versions.bun` and the inherited `npm_config_user_agent` env var — and routes to `bun` equivalents when Bun is the active package manager. The husky hook falls back to `bun` only when `npm` isn't on PATH.

## Specific references

- `.husky/pre-commit` L1–L3: `if command -v npm >/dev/null 2>&1; then PM=npm; else PM=bun; fi`. The fallback prefers `npm` whenever it's available, which is the right default since most CI/dev environments have it. Bun-only contributors get the bun path.
- `scripts/build.js` L24–L31: detection uses `process.versions.bun || npm_config_user_agent.includes('bun')`. The double-check is well-motivated by the inline comment — `bun run build` invokes `node scripts/build.js` per `package.json`, so `process.versions.bun` is unset inside the child despite the build being Bun-driven. The `user_agent` env var is the standard signal package managers set; this is the canonical way to detect this.
- `scripts/build.js` L40–L60: the Bun branch builds `@google/gemini-cli-core` first (everyone depends on it) before fanning out the rest with `bun run --filter '*' --filter '!@google/gemini-cli-core' --parallel build`. The comment explaining "bun run --filter does not respect topological order" is exactly the right context to leave behind. Confirm that no other workspace has a hard build dep on a non-core sibling — if so, the parallel fan-out would race.
- `scripts/build.js` non-CI branch (after the new Bun block): preserved unchanged. Good — Bun path is purely additive.
- `scripts/build_package.js` L35–L48: same `isBun` detection re-implemented locally with a `// See scripts/build.js for why both checks are needed.` comment. Slightly DRY-violating but acceptable; promoting to a shared `scripts/_pm.js` helper would be a follow-up.

## Verdict

verdict: merge-after-nits

## Reasoning

Useful contributor-experience fix with a clear comment trail explaining the non-obvious env-var check. Two pre-merge nits: (1) extract the duplicated `isBun` detection into a tiny shared helper to avoid drift between the two scripts; (2) document the topological-order assumption — if a non-core workspace later grows a sibling dep, the parallel fan-out at `scripts/build.js` L55 will produce flaky builds and the failure mode (a missing artifact during `tsc --build`) won't obviously point back here.
