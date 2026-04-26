# BerriAI/litellm #26539 — chore(deps): bump gitpython from 3.1.46 to 3.1.47

- **Author:** app/dependabot (bot)
- **Head SHA:** `07956859dd4af140be4f3709de981099fb89125b`
- **Base:** `litellm_internal_staging`
- **Size:** +4 / -4 (1 file: `uv.lock`)
- **URL:** https://github.com/BerriAI/litellm/pull/26539

## Summary

Dependabot bumps the transitive Python dependency `gitpython` from
3.1.46 → 3.1.47 in `uv.lock`. Single-file lockfile-only change; no
`pyproject.toml` constraint change because `gitpython` is not a direct
dependency — it's pulled in transitively (likely via a tracing/observability
package). PR also re-stamps `exclude-newer` because uv re-snapshots the
lockfile timestamp on every regeneration.

## Specific findings

- `uv.lock:12` — `exclude-newer = "2026-04-23T02:32:27.506663Z"` →
  `"2026-04-23T02:46:01.651337441Z"`. This is the `uv lock`-generated
  timestamp telling resolution what to consider "new"; the bump is
  ~13 minutes forward, which is consistent with a single targeted
  `uv lock --upgrade-package gitpython` invocation. Not a behavior
  change for installers.
- `uv.lock:1730` — `version = "3.1.46"` → `"3.1.47"` for the
  `gitpython` package entry.
- `uv.lock:1735` — sdist URL and SHA256 swapped:
  `400124c7d0ef4ea03f7310ac2fbf7151e09ff97f2a3288d64a440c584a29c37f` →
  `dba27f922bd2b42cb54c87a8ab3cb6beb6bf07f3d564e21ac848913a05a8a3cd`.
  Format and structure unchanged.
- `uv.lock:1737` — wheel URL and SHA256 swapped:
  `79812ed143d9d25b6d176a10bb511de0f9c67b1fa641d82097b0ab90398a2058` →
  `489f590edfd6d20571b2c0e72c6a6ac6915ee8b8cd04572330e3842207a78905`.
- `upload-time` fields are timestamped 2026-04-22, matching gitpython's
  PyPI release record for 3.1.47.
- Upstream gitpython 3.1.47 is a patch release (Git symbolic-ref edge
  cases, type-hint fixes). No public API removed/renamed. litellm uses
  gitpython only at telemetry bootstrap to record the running commit
  SHA in observability traces — no surface area sensitive to patch-level
  changes.

## Verdict

`merge-as-is`

## Reasoning

Lockfile-only patch bump from a trusted bot, single transitive package.
Hashes and URLs are coherent with PyPI's published 3.1.47 record. No
direct constraint change, no Python source touched. Standard "merge if
green CI" call.
