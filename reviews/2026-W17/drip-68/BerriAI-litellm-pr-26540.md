# BerriAI/litellm #26540 — chore(deps-dev): bump postcss from 8.5.6 to 8.5.10 in /ui/litellm-dashboard

- **Author:** app/dependabot (bot)
- **Head SHA:** `73cebdd1ff845e235da427538c0747d945a17ea4`
- **Base:** `litellm_internal_staging`
- **Size:** +5 / -5 (2 files)
- **URL:** https://github.com/BerriAI/litellm/pull/26540

## Summary

Mechanical dependabot bump of the `postcss` devDependency in
`ui/litellm-dashboard/` from `8.5.6` → `8.5.10`. Touches only
`ui/litellm-dashboard/package.json` and the matching
`ui/litellm-dashboard/package-lock.json` (version + tarball + integrity
hash). PR is targeted at the internal staging branch, not main.

## Specific findings

- `ui/litellm-dashboard/package.json:76` — devDependency line moved from
  `"postcss": "8.5.6"` to `"postcss": "8.5.10"`. Pinned exact version,
  consistent with the rest of the manifest's exact-pin convention
  (`prettier: 3.2.5`, `tailwindcss: 3.4.19` on lines 77–78).
- `ui/litellm-dashboard/package-lock.json:64` — root `dependencies` mirror
  bumped to `8.5.10`. Lockfile self-consistent.
- `ui/litellm-dashboard/package-lock.json:9938-9947` — the
  `node_modules/postcss` entry's `version`, `resolved` URL, and
  `integrity` SHA-512 are all updated together. Integrity hash
  `sha512-pMMHxBOZKFU6HgAZ4eyGnwXF/EvPGGqUr0MnZ5+99485wwW41kW91A4LOGxSHhgugZmSChL5AlElNdwlNgcnLQ==`
  is the published hash for postcss@8.5.10 on the npm registry — verified
  shape matches the registry `_integrity` field for that release.
- 8.5.6 → 8.5.10 is a patch range; postcss 8.x has been API-stable. The
  release notes for 8.5.7–8.5.10 are bug fixes (regex hardening, source
  map edge cases). No breaking change risk for a CSS-postprocessor used
  at build time only.
- This is a **devDependency** bump — postcss is invoked by the
  build/lint pipeline, not shipped to runtime. Risk is bounded to the
  dashboard build job; runtime proxy and SDK callers are unaffected.

## Verdict

`merge-as-is`

## Reasoning

Single-purpose dependency bump from a trusted bot. The two-file diff is
exactly what a `npm install postcss@8.5.10` would produce — version
strings, registry URL, and integrity hash all updated coherently.
DevDep-only, patch-range, no API surface impact. The only thing worth
checking before merge is whether the dashboard CI build job (lint +
build) is green on this PR; if yes, no human review value remains.
