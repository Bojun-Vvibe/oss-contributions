# google-gemini/gemini-cli #26458 — chore/release: bump version to 0.42.0-nightly.20260504.g37edd1d4d

- **Head SHA:** `441de72f970d854de4639e9ae06efe8a4bdc76df`
- **Base:** `main`
- **Author:** gemini-cli-robot (bot)
- **Size:** +19 / −19 across 8 files (root `package.json`, `package-lock.json`, and 6 workspace `package.json` files under `packages/{a2a-server,cli,core,devtools,sdk,test-utils,vscode-ide-companion}`)
- **Verdict:** `merge-as-is`

## Summary

Automated nightly version bump. Mechanical replacement of
`0.42.0-nightly.20260428.g59b2dea0e` →
`0.42.0-nightly.20260504.g37edd1d4d` across all workspace
`package.json` files, the root `package-lock.json`, and the
`sandboxImageUri` config field that pins the Docker sandbox tag.

## What's right

- **Single-string mechanical change, fully symmetric.** Every
  `version` field across the 7 `package.json` files plus the matching
  `version` entries inside `package-lock.json` (lines 4, 9, 18079,
  18208, 18356, 18667, 18682, 18713, 18745) all advance to the
  identical new string. No partial bumps that would leave the
  workspace in a state where `packages/cli` thinks it depends on
  `0.42.0-nightly.20260428.g59b2dea0e` of `packages/core` while
  `packages/core` declares itself as `0.42.0-nightly.20260504...`.

- **`sandboxImageUri` is bumped in lockstep** in both `package.json:17`
  (root) and `packages/cli/package.json:30` to
  `us-docker.pkg.dev/gemini-code-dev/gemini-cli/sandbox:0.42.0-nightly.20260504.g37edd1d4d`.
  This matters: the `sandboxImageUri` is what the runtime pulls when
  the user enables sandbox mode, and if it lags the package version
  the CLI silently runs in a sandbox built from an older snapshot.
  Lockstep is correct.

- **Git SHA suffix `g37edd1d4d` matches the release-build convention**
  (`g` prefix + 9-char short SHA) — the bot produces these from
  `git describe`, so the suffix is verifiable against the actual
  release commit.

- **No code touched.** Pure metadata. Risk surface is "did the lockfile
  hash sums get regenerated incorrectly?" — and since the diff shows
  only `version` field changes (no `resolved`/`integrity` field
  rewrites), the answer is no, lockfile semantics are preserved.

- **Bot author + standard format means the downstream release pipeline
  will accept this PR without manual annotation.** The CI gating that
  publishes `@google/gemini-cli@0.42.0-nightly.20260504.g37edd1d4d` to
  npm reads the version field; this PR satisfies that contract by
  construction.

## Nits (non-blocking)

- The PR body is a single sentence ("Automated version bump for
  nightly release.") which is fine for a bot PR but means a
  reviewer can't immediately verify *which* base commit
  `g37edd1d4d` corresponds to. A two-line bot enhancement would
  help: include the resolved upstream SHA + the diff range
  (`compare/g59b2dea0e...g37edd1d4d`) so reviewers can sanity-check
  what's being released. Not a reason to block this PR.

- `package-lock.json` v3 lockfile shows only `version` field updates,
  which is correct for a workspace-internal version bump (no resolved
  URL or integrity hash should change because no remote tarball
  exists for these workspace-`file:` packages). Quick gut-check that
  the bump generator preserved this invariant — confirmed by the
  hunks shown, no `integrity:` lines mutated.

## Risks

- **Effectively zero.** This is a non-functional metadata advance.
  The only failure modes are (a) version-mismatch between workspace
  packages — ruled out by the symmetric diff, (b) sandboxImageUri
  drift — ruled out by lockstep bump, (c) lockfile corruption —
  ruled out by `version`-only edits.

- **One tiny vigilance item:** if any consumer of these packages
  outside this repo pins to `0.42.0-nightly.20260428.g59b2dea0e`
  exactly (rather than a `^0.42.0-nightly` range), this bump would
  not affect them either way — they'd just keep getting the old
  version until they update. Standard nightly-release semantics; not
  a problem with this PR.

## Verdict reasoning

`merge-as-is`: this is exactly the kind of bot PR that should land on
green CI without ceremony. Symmetric metadata bump, sandboxImageUri
preserved-in-lockstep, no code touched, no lockfile semantics
violated. The release pipeline gates further safety downstream.
