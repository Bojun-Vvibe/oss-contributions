---
pr: 14131
repo: All-Hands-AI/OpenHands
sha: 219407069b9db0b2e85c0f8479f5ab78acdcaa4f
verdict: merge-as-is
date: 2026-04-26
---

# All-Hands-AI/OpenHands #14131 — chore(deps): bump the security-all group across 1 directory with 7 updates

- **URL**: https://github.com/All-Hands-AI/OpenHands/pull/14131
- **Author**: dependabot[bot]
- **Head SHA**: 219407069b9db0b2e85c0f8479f5ab78acdcaa4f
- **Size**: +229/-248, 1 file (`poetry.lock`)

## Scope

Grouped dependabot security bump under the `security-all` group. Lockfile-only change — no `pyproject.toml` modifications, so all 7 packages either had their existing version constraints already permitting the new pin, or are transitive deps. Per the PR body:

| Package | From | To |
| --- | --- | --- |
| authlib | 1.6.9 | 1.6.11 |
| poetry | 2.3.3 | 2.3.4 |
| pypdf | 6.9.2 | 6.10.2 |
| jwcrypto | 1.5.6 | 1.5.7 |
| lupa | 2.6 | 2.7 |
| lxml | 6.0.2 | 6.1.0 |
| nbconvert | 7.17.0 | 7.17.1 |

## Specific findings

- **The headline fix is `authlib 1.6.9 → 1.6.11`,** which patches **CVE-style CSRF in the Starlette OAuth client** (per the release notes embedded in the PR body): "Fix CSRF vulnerability in the Starlette OAuth client when a `cache` is configured." OpenHands uses Starlette/FastAPI in `enterprise/server.py` and several auth callbacks, so this is directly relevant. 1.6.10 also patched a related issue: "Fix redirecting to unvalidated `redirect_uri` on `UnsupportedResponseTypeError`." Both are real auth-surface fixes, not theoretical.

- **`jwcrypto 1.5.6 → 1.5.7`** — JOSE/JWT crypto library, security-grouped by dependabot. Within-major patch, low risk of API drift. The lockfile diff at `poetry.lock:4425-4440` shows only the version + hash change plus a cosmetic `typing-extensions` → `typing_extensions` normalization in the dep declaration (PEP 503 underscore/hyphen normalization, semantically identical).

- **`lupa 2.6 → 2.7`** — Python wrapper around Lua/LuaJIT. The lockfile diff is large (lines 4925+ show ~50+ wheel-hash entries replaced) but that's normal for binary-wheel packages: each Python-version × OS × arch combo gets its own wheel hash. Notably `python-versions = "*"` → `python-versions = ">=3.8"` in the metadata, which is a tightening of supported Pythons, but OpenHands targets `>=3.12` per its `pyproject.toml`, so no impact.

- **`pypdf 6.9.2 → 6.10.2`** — within-major bump on a PDF parser. Active CVE history in this package family makes the security label plausible (PDF parsers regularly get malformed-input DoS or memory-corruption advisories). Within `6.x` minor jumps are usually safe.

- **`lxml 6.0.2 → 6.1.0`** — minor bump. lxml is a heavy native-extension dep used widely; version pins on it tend to be sticky. Within-major minor bump should be safe; the upstream changelog usually highlights any C-API or namespace-handling regressions and 6.0→6.1 is unlikely to break consumers.

- **`poetry 2.3.3 → 2.3.4`** — patch on the build tool itself. Affects only the developer environment (poetry isn't shipped to runtime), so impact is "developers will get the patched version next `poetry install`." Zero runtime risk.

- **`nbconvert 7.17.0 → 7.17.1`** — patch on the Jupyter notebook converter. Only matters if OpenHands actually exercises the nb-conversion path; looking at the package, it's likely a transitive of a notebook-rendering dep. Patch-level bump = safe.

- **Diff is `poetry.lock` only.** No `pyproject.toml` or `pyproject.toml`-shaped manifest change. That means all 7 bumps are within their existing constraint windows — dependabot didn't have to relax any version pins to make this work. That's the cleanest possible shape for a security-group bump and the lowest-friction path to merge.

- **Author is dependabot[bot].** The repo's CI pipeline (poetry install + pytest + lint matrix) is the right gate here; if it goes green, this should land. Manual review beyond "skim release notes for breaking changes" is overkill for grouped within-major security bumps.

## Risk

Low. The specific surfaces to watch:
1. **authlib's Starlette OAuth client** — if OpenHands has tests that mock the cached-CSRF flow, they should still pass; if there's no test coverage there, manually exercise the OAuth login post-merge once.
2. **lxml** — heavy native dep, occasional ABI surprises. CI's `pip install` step would catch wheel-resolution issues.
3. **lupa** — only matters if any OpenHands path actually executes Lua (unlikely, but the dep is in the tree for some reason).

## Verdict

**merge-as-is** — grouped security bump with one real OAuth/CSRF fix (authlib 1.6.11) and 6 patch/minor housekeeping bumps. Lockfile-only, no manifest changes, all within-major. CI green = ship. The auth-surface fix alone justifies prioritizing this over a slow-roll review.

## What I learned

`authlib`'s 1.6.10 + 1.6.11 patches together fix two adjacent OAuth-flow bugs (open-redirect on error response + CSRF when caching is on). When a security release lands two patches in close succession on the *same* code area, the right move is to pick up both — taking only 1.6.10 leaves the CSRF window open. Worth treating "two consecutive security patches on adjacent code paths" as a special case: don't pin to the lower one, jump to the latest. Dependabot's grouped security PRs do this automatically; manual pinning often doesn't.
