# BerriAI/litellm #27157 — [Infra] Bump deps

- **Head SHA:** `c70a047eb905908e009bc7434c85c1fa86e1d25f`
- **Author:** @yuneng-berri
- **Verdict:** `merge-after-nits`
- **Files:** +8 / −8 across `enterprise/pyproject.toml`, `litellm-proxy-extras/pyproject.toml`, `pyproject.toml`, `uv.lock`

## Rationale

Mechanical version bump for two internal sub-packages: `litellm-proxy-extras` 0.4.70 → 0.4.71 and `litellm-enterprise` 0.1.39 → 0.1.40. The diff is exactly what you'd expect from a coordinated `cz bump` across two cz-managed sub-packages: each `pyproject.toml` updates both the `[project]` `version` line and the `[tool.commitizen]` `version` line (so subsequent `cz` runs stay synchronized), the parent `pyproject.toml:55-56` pins both bumped versions in the `proxy` extra dependency list, and `uv.lock:3417,3422` mirrors the new versions for both editable sub-packages. All four edits are internally consistent — no version skew between the four files, no stale reference left at an older version.

Two real concerns, both minor:

1. **Empty `## Changes` section in the PR body.** The PR template's `## Changes` heading is left blank. For an internal-package version bump, the reviewer needs to know what's actually inside `0.4.70 → 0.4.71` and `0.1.39 → 0.1.40` — i.e., what behavioral or dependency changes those sub-package bumps drop into the parent proxy. The bumps are by definition opaque from the parent diff (no `pip install -e ./litellm-proxy-extras` content is visible here), so a reviewer would need to either go read the sub-package CHANGELOG entries or assume zero behavior change. Three lines in the PR body listing the actual deltas (e.g., "litellm-proxy-extras 0.4.71: bumps Prisma client to X / adds Y migration / no schema change") would let this land confidently. Without it, the maintainer is being asked to trust the version-bump commit messages in the sub-packages, which the PR doesn't link.

2. **Pre-Submission checklist all unchecked.** Specifically the test checkbox ("I have Added testing in `tests/test_litellm/`"). A version bump doesn't *need* a new test — that's accepted CI practice — but the PR also doesn't link the CI run for the bumped versions on this branch. The template explicitly asks for "Branch creation CI run" / "CI run for the last commit" / "Merge / cherry-pick CI run" links and they're all blank. For an internal-only `litellm-proxy-extras` / `litellm-enterprise` bump where the maintainer can't easily preview behavior, the green-CI link is the practical guarantee that the new versions don't break the proxy boot path or the enterprise-feature load path.

The bump itself is fine and the four-file change is internally consistent — minor (`0.1.39→0.1.40`, `0.4.70→0.4.71`) version segments in both packages signal intent of "no breaking change", and the parent's pinned `==` constraint at `pyproject.toml:55-56` correctly forces lockstep so an old proxy can't accidentally pick up the new sub-package or vice versa. Land it once the PR body lists the actual sub-package deltas and the green-CI link is attached. If both come back clean, this is a one-tick merge.
