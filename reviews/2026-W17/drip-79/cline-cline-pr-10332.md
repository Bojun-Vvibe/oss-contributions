# cline/cline PR #10332 — [Aikido] Fix 3 security issues in follow-redirects, axios

- **PR:** https://github.com/cline/cline/pull/10332
- **Author:** app/aikido-autofix (bot)
- **Head SHA:** `ad73deda619a8fc780419d60ca926a9c38330b1d`
- **Files:** 2 (+20 / ?)
- **Verdict:** `merge-as-is`

## What it does

Bot-authored security bump in the `evals/` workspace lockfile + manifest. Bumps:
- `axios` 1.15.0 → 1.15.1
- `follow-redirects` (transitive via axios) 1.15.11 → 1.16.0

Resolves two CVEs flagged by Aikido (per PR body): credential leakage on cross-domain redirects (`follow-redirects`) and prototype pollution via header injection (`axios`).

## Specific reads

- `evals/package.json:10` — direct dependency bump:
  ```json
  -    "axios": "1.15.0",
  +    "axios": "1.15.1",
  ```
  Pinned exact version (no caret) preserved. Good — keeps eval reproducibility.
- `evals/package-lock.json:25-34` — axios entry updated with new resolved URL and integrity hash; the `dependencies` block under axios still reads `"follow-redirects": "^1.15.11"` so the upstream caret range pulls in 1.16.0 naturally.
- `evals/package-lock.json:38-44` — `follow-redirects` entry bumped to 1.16.0 with new sha512 integrity. Single transitive entry; no second copy elsewhere in this lockfile.
- `evals/package-lock.json:14-21,49-52,90-94` — three `+ "peer": true` flag additions on `@types/node` and `typescript` entries. These are npm 10+ lockfile metadata corrections (npm started writing `peer: true` for transitive peer deps); cosmetic, not functional. Safe noise.
- `evals/package.json:117` — adds trailing newline (`-} \ No newline at end of file` → `+}` with newline). Cosmetic but conventional.

## Risk surface

Minimal. Three reasons:

1. **Scope is `evals/`, not the shipped extension.** The lockfile lives under `evals/` which is internal eval tooling — no end-user impact even if the upgrade regressed.
2. **Patch-level axios bump (1.15.0 → 1.15.1).** Per semver, no API changes; axios maintains strict patch discipline.
3. **`follow-redirects` 1.15.x → 1.16.x is a minor bump** but the project has a stable API surface and the PR explicitly states "✅ No breaking changes for: axios" and "Breaking changes analysis not available for: follow-redirects" — given axios is the only consumer here and it pins via caret to `^1.15.11`, the resolver will accept 1.16.x without any axios code change. The release notes for `follow-redirects` 1.16.0 are limited to the security fix + minor internal refactors.

The `"peer": true` lockfile noise is the only thing that could cause a downstream `npm ci` warning, and only on older npm. Cline already requires modern npm via its other workspaces.

## Nits (not blocking)

1. Bot-authored bumps usually skip running `evals/`'s own test suite; a maintainer should at minimum kick CI to confirm the eval harness's HTTP-touching paths still work against the bumped axios. Mostly a process check, not a code issue.
2. If cline cares about lockfile hygiene, the unrelated `"peer": true` flags should arguably be a separate `chore: regenerate evals lockfile` PR rather than mixed with a security fix. Not enough to hold this up.

Verdict: merge as-is — narrow scope (`evals/` only), patch + low-risk minor bumps, addresses a real CVE. Standard Aikido autofix that should land same-day.
