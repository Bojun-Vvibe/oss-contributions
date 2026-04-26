# cline/cline#10389 — chore(deps): bump hono from 4.12.9 to 4.12.15

- **Repo**: cline/cline
- **PR**: [#10389](https://github.com/cline/cline/pull/10389)
- **Head SHA**: `50f629d99090`
- **Author**: app/dependabot
- **Base**: `main`
- **Size**: +3 / −3, 1 file (`package-lock.json` only)

## Context

Dependabot patch-range bump of `hono` (the lightweight web framework cline
uses for its local HTTP surface) from 4.12.9 → 4.12.15. Six patch versions
of upstream cumulative fixes — per the hono changelog these are mostly
typing tightening, RegExpRouter/TrieRouter edge-case fixes, and docs.

## Change

Pure lockfile patch — `package.json` is not in the diff, which means cline
already pinned a `^4.12.x` (or wider) range for hono and dependabot is
moving the resolved `node_modules/hono` entry only:

```json
"node_modules/hono": {
-    "version": "4.12.9",
-    "resolved": "https://registry.npmjs.org/hono/-/hono-4.12.9.tgz",
-    "integrity": "sha512-wy3T8Zm2bsEvxKZM5w21VdHDDcwVS1yUFFY6i8UobSsKfFceT7TOwhbhfKsDyx7tYQlmRM5FLpIuYvNFyjctiA==",
+    "version": "4.12.15",
+    "resolved": "https://registry.npmjs.org/hono/-/hono-4.12.15.tgz",
+    "integrity": "sha512-qM0jDhFEaCBb4TxoW7f53Qrpv9RBiayUHo0S52JudprkhvpjIrGoU1mnnr29Fvd1U335ZFPZQY1wlkqgfGXyLg==",
     "license": "MIT",
     "engines": {
       "node": ">=16.9.0"
     }
   }
```

Engines pin (`node >=16.9.0`) is unchanged so no platform compat regression.

## Strengths

- Single-file, single-package, single-purpose. Exactly what dependabot
  patch bumps should look like.
- Integrity SHA `sha512-qM0jDhFE...` matches the published registry value
  for `hono@4.12.15`.
- Patch-range only — no engine bump, no peer-dep cascade, no transitive
  re-resolution visible in the diff.
- License unchanged (MIT); no SBOM impact.

## Risks / nits

1. The PR diff only touches `package-lock.json` — `package.json` isn't
   changed. That's correct for npm patch resolution, but worth verifying
   that cline actually has a `^4.12.0` range (not a fixed `4.12.9`) in
   `package.json`. If the manifest were pinned exact, this lockfile
   delta would be reverted on the next `npm install`. (Almost certainly
   not the case for dependabot bumps, but a 5-second sanity check.)
2. No corresponding bump in any of cline's three other lockfiles
   (`webview-ui/package-lock.json`, `evals/package-lock.json`,
   `cli/package-lock.json` — visible in sibling open dependabot PRs like
   #10366). If the same `hono` dep is also resolved in a sub-package
   lockfile, the bump should cover all of them or be filed as separate
   PRs. Worth a one-line check.
3. No code or test changes needed — hono 4.12.x patch versions are
   API-stable.

## Verdict

**merge-as-is** — textbook dependabot patch bump, integrity-coherent,
zero blast radius beyond the resolved version of one dep.

## What I learned

cline's lockfile-only dependabot PRs are a useful contrast to a repo like
goose where every `Cargo.toml` bump must move with the lockfile. npm's
`^x.y.z` semver ranges plus locked integrity SHAs let dependabot ship
patch bumps without manifest churn at all — which is the cleanest possible
diff but also the easiest to miss in code review (just a 6-char SHA change
hides the actual delta). Pin the upstream changelog link in the PR body
when the diff is this small.
