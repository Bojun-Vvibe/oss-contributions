# BerriAI/litellm #26961 — [Infra] Bump Versions

- **PR**: https://github.com/BerriAI/litellm/pull/26961
- **Head SHA**: `6da13efcec80`
- **Files reviewed**: `litellm-proxy-extras/pyproject.toml`, `pyproject.toml`, `uv.lock`
- **Date**: 2026-05-01 (drip-229)

## Context

Mechanical version bump for the `litellm-proxy-extras` companion
package (0.4.69 → 0.4.70) plus the matching pin in the main
`pyproject.toml` and the regenerated `uv.lock` entry. This is the
release-train coordination shape: the main `litellm` package's
`proxy` extra hard-pins the extras version (`==0.4.69` previously), so
any extras release requires a follow-up bump here to make `pip install
litellm[proxy]` resolve to the new wheel.

## Diff (3 files, +4 -4)

`litellm-proxy-extras/pyproject.toml:3`:

```diff
-version = "0.4.69"
+version = "0.4.70"
```

Plus the same flip at `:29` (commitizen `version` mirror).

`pyproject.toml:55`:

```diff
-    "litellm-proxy-extras==0.4.69",
+    "litellm-proxy-extras==0.4.70",
```

`uv.lock:3421`:

```diff
-version = "0.4.69"
+version = "0.4.70"
```

## Observations

1. **Three-way pin is consistent.** All three of (a) the extras
   package's own `[project] version`, (b) the commitizen `[tool.commitizen]
   version` mirror used by the release-bot, and (c) the main
   `pyproject.toml`'s hard-pin in the `proxy` optional-deps extra now
   read `0.4.70`. The `uv.lock` regeneration matches. Missing any one
   of the three is a known foot-gun for this repo (the commitizen
   mirror in particular drifts silently if the bumper script is run
   manually rather than via the bot).

2. **Hard pin (`==0.4.70`) preserved.** The main `pyproject.toml`
   keeps the `==` specifier rather than `>=` / `~=`. That's
   intentional given how tightly the extras ship in lockstep with the
   main package — a `>=` would let `pip` resolve to a future extras
   that has API drift from the installed `litellm` core.

3. **No source change.** The PR carries only version-string flips and
   the lockfile regeneration. There is no behavior to review beyond
   the consistency of the three pin sites.

## Nits

- **No CHANGELOG / release notes for the 0.4.70 extras.** Hard to
  tell from this diff alone what behavior changed in the extras
  package between `.69` and `.70` — readers downstream consuming the
  bump would have to dig into the extras subdirectory's commit log.
  A one-line note in the PR body ("0.4.70 ships X, Y, Z") would
  reduce that friction. Not a code-change blocker.

- **No test/CI assertion that the three pins agree.** This same shape
  has been bumped many times across the project's history; a tiny
  CI guard (a one-line script that diffs the three version strings
  and fails if they disagree) would prevent the recurring
  "forgot the commitizen mirror" / "forgot the main pyproject pin"
  drift class. Worth a follow-up issue rather than blocking this PR.

## Verdict

`merge-after-nits` — purely mechanical version bump with all three
canonical pin sites correctly updated and lockfile regenerated. The
nits are out-of-band (release notes, CI guard) and shouldn't block
the train.
