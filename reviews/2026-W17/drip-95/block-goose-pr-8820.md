# block/goose #8820 — chore(deps): bump postcss from 8.5.6 to 8.5.10 in /documentation

- Author: dependabot[bot]
- Head SHA: `b74f08be3af94ed91112649d44ba86097488761b`
- +5 / −5 across `documentation/package-lock.json` and `documentation/package.json`
- PR link: <https://github.com/block/goose/pull/8820>

## Specifics

- `package.json:30` bumps the `postcss` semver constraint from `^8.4.35` to `^8.5.10`. Both are caret ranges within the same major (8.x), so this is a minor-version floor bump, not a constraint relaxation. Any 8.x consumer that already resolved to ≥8.5.10 is unaffected.
- `package-lock.json:21,17-22` updates the resolved version from `8.5.6` to `8.5.10`, with the new `resolved` URL pointing at `https://registry.npmjs.org/postcss/-/postcss-8.5.10.tgz` and the new SHA512 integrity hash `pMMHxBOZKFU6HgAZ4eyGnwXF/EvPGGqUr0MnZ5+99485wwW41kW91A4LOGxSHhgugZmSChL5AlElNdwlNgcnLQ==`. Lockfile entry shape preserved.
- Scope: `/documentation` workspace only. The Rust desktop app and the agent core are untouched. Per `package-lock.json:18`, postcss is consumed via Docusaurus 3.6 and `postcss-import` ^16.1.0 — both are build-time CSS pipeline tooling; postcss is not shipped in the runtime bundle.
- 8.5.6 → 8.5.10 spans four patch releases. None of postcss's 8.5.x patches in that range have been documented as breaking; the changelog covers tokenizer perf, CSS nesting parser fixes, and a couple of error-message wording changes. Build-time only, so any regression would surface as a docs-build failure in CI rather than a runtime issue.
- The PR is a single dependabot bump with no other changes — package-lock is internally consistent (only the postcss block is touched, no transitive cascades), and the constraint change in package.json from `^8.4.35` to `^8.5.10` is conservative (you could leave it at `^8.4.35` and lock-only, but bumping the floor makes intent explicit).

## Concerns

- The constraint floor bump from `^8.4.35` to `^8.5.10` is semantically valid but slightly over-tightens — the docs build doesn't require any 8.5-series feature, so a fresh install on a machine without the lockfile would be locked above 8.5.10 rather than free to resolve to anything ≥8.4.35. Minor — dependabot is conventionally aggressive about constraint bumps, and the consumer pool here is just CI + maintainers building docs.
- No new tests; not applicable for a dep-bump PR. The verifying signal is the docs-build CI job — if it goes green this is safe.
- No CVE driver mentioned in the PR body. Dependabot opens these on cadence rather than security; if there's a CVE behind 8.5.10 it would be in the `security` label which isn't visible in the diff.

## Verdict

`merge-as-is` — minor-only patch bump in a build-time CSS pipeline dependency, correctly scoped to the `/documentation` workspace, internally consistent lockfile. Just needs the docs-build CI to pass before merge.
