---
pr: 10366
repo: cline/cline
sha: 6e6bdeeb037785570904a5cd1d4a06fc89b784e8
verdict: merge-as-is
date: 2026-04-26
---

# cline/cline #10366 — chore(deps): bump the npm_and_yarn group across 4 directories with 11 updates

- **URL**: https://github.com/cline/cline/pull/10366
- **Author**: dependabot[bot]
- **Head SHA**: 6e6bdeeb037785570904a5cd1d4a06fc89b784e8
- **Size**: +2891/-436 across 6 lockfile/package.json files in `docs/`, `evals/`, `evals/analysis/`, `testing-platform/`

## Scope

Grouped security bump touching 4 ancillary directories (none of them the main extension source):

- `docs/` — `lodash 4.17.21 → 4.18.1`
- `evals/` — `follow-redirects` bump (lockfile-only resolution)
- `evals/analysis/` — `esbuild` bump + `package.json` direct dep change
- `testing-platform/` — `lodash 4.17.21 → 4.18.1` + `package.json` direct dep change

The PR is dependabot-grouped under `npm_and_yarn` security advisories. The biggest line-count is `evals/analysis/package-lock.json` at +1874/-1, which is just transitive churn from one direct esbuild bump.

## Specific findings

- **The lodash 4.18.1 bump is the headline.** Per the dependabot release-notes block in the PR body, `lodash 4.18.0` patched two real CVEs:
  - **CVE-2026-4800** / GHSA-r5fr-rjxr-66jc — `_.template` code injection via `imports` keys (incomplete patch for CVE-2021-23337). The original CVE-2021-23337 was fixed by guarding the `variable` option against `reForbiddenIdentifierChars`, but `importsKeys` was left unguarded; the new release blocks it too.
  - GHSA-f23m-r3pf-42rh — `_.unset` / `_.omit` prototype pollution via `constructor`/`prototype` path traversal. Array-wrapped path segments and primitive roots could bypass earlier guards. Now `constructor` and `prototype` are blocked unconditionally as non-terminal path keys.

  **4.18.1** is then a tiny patch on top of 4.18.0 that fixes a `ReferenceError` regression in the `template` and `fromPairs` modular builds, so taking 4.18.1 instead of 4.18.0 is the right call.

- **None of these dirs ship to end users of the cline extension.** `docs/`, `evals/`, `evals/analysis/`, and `testing-platform/` are all developer-tooling adjuncts — eval harness, docs site, internal testing scaffolding. The main `package.json` for the extension itself isn't touched. So the worst-case impact of a regression is "cline maintainers' eval workflow breaks," not "shipped extension breaks."

- **Risk concentration is in the `evals/analysis/` esbuild bump,** which produces +1874/-1 of lockfile churn — a major-version esbuild bump can shift bundle output subtly (different minification, different tree-shaking edges). Maintainer should verify the eval-analysis tooling still produces the expected outputs after this lands. For `lodash 4.17.21 → 4.18.1`, that's a within-major bump and the API surface used (presumably `_.template`/`_.unset`/`_.omit`/etc.) is unchanged for normal calls; the only behavior change is "code that *was* exploiting the prototype-pollution path now returns `false` instead of mutating," which is exactly the desired behavior.

- **`follow-redirects` lockfile-only bump** — also a known security-advisory class crate (HTTP redirect handling, history of credential-leak CVEs). Lockfile resolution moves are routinely safe within the same major version, and `follow-redirects` is conservative about API breakage.

- **Author is dependabot[bot].** Standard automated grouped PR. Cline's policy presumably auto-merges grouped security PRs that touch only ancillary dirs after CI passes. This is exactly that shape.

- **No `package.json`-level `package-lock.json` for the extension itself in the diff.** Confirms the user-facing extension dependency tree is untouched. This PR is "internal tooling hygiene + advertised security bumps" — the bar for landing it should be "CI green," not "deep manual review."

## Risk

Low. The only meaningful surface is the esbuild bump in `evals/analysis/`, which could affect the eval-analysis bundle. Worst case: maintainer reruns evals after merge and finds output drift; revert is one commit. Lodash and follow-redirects are within-major patches with security backing — safer to land than to delay.

## Verdict

**merge-as-is** — grouped dependabot bump on developer-tooling dirs, with two real CVE fixes (lodash prototype pollution + `_.template` code injection) and one HTTP-redirect security crate refresh. CI green is sufficient signal here. The cline extension itself isn't affected.

## What I learned

`lodash 4.18.1` is the new floor for any project that uses `_.template` with attacker-influenced `imports` keys, or `_.unset`/`_.omit` with attacker-influenced paths — both have *new* CVEs in 2026 on top of the 2021 ones. Worth grepping every internal codebase for `lodash` < 4.18.1 and `_.template`/`_.unset`/`_.omit` callsites, even if they're "internal-only," because the real-world boundary between "internal" and "user-controlled input flowing into a template render" is usually thinner than people remember.
