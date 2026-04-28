# BerriAI/litellm #26677 — fix(ui): correct npm min-release-age values

- PR: https://github.com/BerriAI/litellm/pull/26677
- Head SHA: `879d79463057`
- Diff: 6+/6- across 6 `.npmrc` files
- Base: `main`

## Verdict: merge-as-is

## Rationale

- **Closes a silent supply-chain-hardening regression.** The `min-release-age=3d` value at every `.npmrc` site is *invalid pnpm syntax* — pnpm's `min-release-age` setting takes a number of *minutes* (per the pnpm docs and the `@pnpm/config` schema), not a `Xd`/`Xh`/`Xs` duration string. When pnpm parses `3d` it fails type validation and falls back to `0`, which means the supply-chain mitigation that drip-9 reviewed (10080-minute / 7-day install-boundary on `pnpm-lock.yaml` directories) is *not* in effect anywhere `min-release-age` was set with the `3d` form. The `3` value (3 minutes — clearly a typo for `4320` = 3 days) at least surfaces as a non-zero release-age check, but the *intent* in the comments at `:1-3` ("Protects local npm install only") plus the prior `3d` value strongly implies the author meant 3 days.
- **Six call sites in lock-step.** The same wrong literal `3d` was duplicated across `.npmrc`, `litellm-js/proxy/.npmrc`, `litellm-js/spend-logs/.npmrc`, `tests/proxy_admin_ui_tests/.npmrc`, `tests/proxy_admin_ui_tests/ui_unit_tests/.npmrc`, and `ui/litellm-dashboard/.npmrc`. The PR fixes all six identically, which is the right shape — partial fixes here would leave some install boundaries unguarded. The byte-identical hash on every diff (`168e81a1c4e..7999681cc35`) confirms the `.npmrc` files were already drift-free copies of one canonical file, and a follow-up could centralize them, but that's out of scope.

## Nits / follow-ups

- **`3` is still wrong if the intent is "3 days".** pnpm's `min-release-age` is in minutes, so `3` = 3 minutes — barely any protection at all against a fast-moving compromise. The drip-9 review of the original `minimumReleaseAge: 10080` (7 days, in minutes) is the precedent. A follow-up should either (a) change `3` → `4320` (3 days in minutes) if the comment in `:3` represents the actual intent, or (b) explicitly document why 3 minutes is the chosen value. The current PR is correct as a syntax fix but worth a follow-up commit to align with the drip-9 hardening pattern.
- **No PR body / commit message explains the unit gotcha.** A one-line note along the lines of "pnpm's min-release-age is in minutes, not duration strings; `3d` was being silently parsed as 0" would help reviewers and the eventual incident-postmortem reader understand why the supply-chain protection was effectively off.
- **The five `.npmrc` files in `litellm-js/` and `tests/` and `ui/` are exact byte-for-byte duplicates of the root `.npmrc`** (same blob hashes both before and after) — consider centralizing into a single canonical file with `npmrc` extends/include directive, or document the duplication intent so future fixes don't have to chase six copies.

## What I learned

`min-release-age` in pnpm and npm-config-style ecosystems is a unit-poisoned setting: pnpm takes minutes, but the broader npm community widely uses duration strings for similar settings (`npm config set fund false` etc.), so the `3d` literal looks reasonable to a Node.js dev but silently fails. The fact that the `.npmrc` parser doesn't error out on type mismatch (it just resolves to `0` and proceeds) is the same fail-open vs fail-loud pattern called out in earlier drips. The fix is one character but the lesson is "supply-chain mitigations need a test that asserts the mitigation is actually in effect" — a CI step that runs `pnpm config get min-release-age` and asserts `> 0` would have caught this on the original drip-9 commit.
