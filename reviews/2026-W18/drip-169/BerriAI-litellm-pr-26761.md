# BerriAI/litellm#26761 — `fix(dashboard): use numeric npm min-release-age`

- PR: https://github.com/BerriAI/litellm/pull/26761
- Head SHA: `309e047a19393bc8f21cf1d881a047f27324a35e`
- Author: Kcstring
- Fixes: #26668

## Summary of change

One-line config fix in `ui/litellm-dashboard/.npmrc`: change
`min-release-age=3d` to `min-release-age=3`. npm 11 parses
`min-release-age` as a *number of days* (integer); the `3d` suffix
syntax — which works for related fields like `cache-min` in older npm
or for some lockfile-tooling configs — is rejected as invalid by npm
11's config validator, so the previous line was being treated as
"unset" and the supply-chain delay window collapsed to zero on
local installs.

## Specific observations

- `ui/litellm-dashboard/.npmrc:5` — `min-release-age=3` is correct
  per `npm help 7 config`: the field expects a non-negative integer,
  unit is days. The companion `min-release-age-exclude` (also in
  npm 11) accepts a comma-separated package list and isn't touched
  here, which is fine for this fix.
- The accompanying comment two lines up (`# Protects local npm
  install only — npm ci (used in CI) ignores this`) is accurate as
  far as it goes, but the *reason* `npm ci` ignores it is that
  `npm ci` consumes `package-lock.json` directly without
  re-resolving releases — worth a one-line elaboration so future
  readers don't try to "fix" CI to honor the same setting.
- Verification steps in the PR description (`npm --version`,
  `npm config get`, `npm config list --location=project`) are the
  right minimum, but none of them prove that the *previous*
  `3d` value was being silently ignored. A reproducer would be
  `npm config get min-release-age` against the old file vs. the new
  one — the old should print `null`/empty + a warning, the new
  should print `3`.
- No corresponding fix anywhere else in the repo. A grep for
  `min-release-age` should be in the PR's verification to confirm
  there isn't a second copy of this config (e.g. in
  `docs/`, `cookbook/`, or examples) that's still using `3d`.
- Pure config change, no runtime code touched, no test impact.
  Cleanest possible follow-up to a pinning policy.

## Verdict

`merge-as-is` — single-line, mechanically correct fix to a config
field that npm 11 was silently rejecting. The followups (grep for
other `3d` instances, reproducer command in the PR body) can land
later or as a comment.
