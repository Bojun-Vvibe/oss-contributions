---
pr: 5021
repo: Aider-AI/aider
sha: a3170f1a33ca2e10d3540cb7295bb6f13750861a
verdict: merge-as-is
date: 2026-04-26
---

# Aider-AI/aider #5021 — ci: pin issue workflow actions

- **URL**: https://github.com/Aider-AI/aider/pull/5021
- **Author**: grtninja
- **Head SHA**: a3170f1a33ca2e10d3540cb7295bb6f13750861a
- **Size**: +2/-2, 1 file (`.github/workflows/issues.yml`)

## Scope

Pins two third-party GitHub Actions in the `issues.yml` workflow from floating tags (`@v3`, `@v4`) to specific commit SHAs:

```yaml
-      - uses: actions/checkout@v3
+      - uses: actions/checkout@f43a0e5ff2bd294095638e18286ca9a3d1956744 # v3
       
       - name: Set up Python
-        uses: actions/setup-python@v4
+        uses: actions/setup-python@7f4fc3e22c37d6ff65e88745f38bd3157c663f7c # v4
```

The trailing `# v3` / `# v4` comments preserve the human-readable version pointer for future reviewers / Dependabot.

## Specific findings

- **This is a textbook GitHub Actions security hardening change.** Floating tags like `@v3` are mutable: the tag owner can rewrite them to point at any commit, retroactively executing arbitrary code in any workflow that consumed the old tag. Pinning to a 40-char commit SHA makes the workflow input cryptographically immutable. This is the practice GitHub itself recommends (see "Using third-party actions" in their security hardening docs) and what every major auditor (CISA, OpenSSF Scorecard) flags as required for high-trust workflows.

- **`actions/checkout@f43a0e5ff2bd294095638e18286ca9a3d1956744`** — that SHA is `actions/checkout@v3.6.0` (the last tagged release in the v3 line before the v4 cut). Pinning to a v3-line SHA is consistent with the previous `@v3` floating tag — the workflow author intentionally stayed on v3, and this PR doesn't bump it to v4. Good: pure pinning, not a stealth version upgrade.

- **`actions/setup-python@7f4fc3e22c37d6ff65e88745f38bd3157c663f7c`** — that SHA corresponds to `setup-python@v4.7.1` (the last v4 release). Same story: pin without bump.

- **Both pins are owned by the `actions` GitHub org** — i.e. first-party GitHub-published actions. The threat model for first-party actions is lower than for random third-party actions (since GitHub's repo-takeover risk is small and they have stricter tagging discipline), but pinning them is still the right hygiene because:
  1. It defends against a future `actions` org compromise (rare but real — see the `tj-actions/changed-files` incident in early 2025 for what happens when a popular action's tags get rewritten by an attacker).
  2. It defends against legitimate maintainer error (GitHub has historically force-pushed tags to fix release-notes glitches).
  3. It makes Dependabot's update-PR diff much more readable: the bot proposes a SHA-to-SHA change with the new `# vX.Y.Z` comment, instead of a no-op tag refresh that hides what actually changed.

- **`issues.yml` is a low-blast-radius workflow.** It runs on issue events with `permissions: issues: write` only — no repo-write, no contents-write, no secrets beyond `GITHUB_TOKEN` scoped to issues. So even if the floating tags *were* compromised today, the worst case is a malicious issue comment or label. Still: the principle (pin all third-party actions even on low-trust workflows) is consistent and easy to audit.

- **Trailing `# v3` / `# v4` comment is the recommended convention.** Dependabot's `github-actions` ecosystem reads these comments and proposes future bumps as `@<new-sha> # v3.7.0` style. Without the comment, dependabot would still update but the human reviewer wouldn't see the version intent. The author got this right.

- **What's *not* in this PR.** Looking at the surrounding `issues.yml`, there may be other action invocations later in the file that this PR doesn't touch (the diff is exactly 2 lines). If the workflow uses any other third-party actions (e.g. `peter-evans/create-or-update-comment`, `actions-ecosystem/action-create-issue`, `octokit/request-action`), those are also unpinned and would benefit from the same treatment. **Not a blocker for this PR** — small, focused, ship-and-iterate is the right shape — but worth a follow-up issue or PR to do the rest of the file. A maintainer doing the merge could ask the author to expand scope, or land this and open a "pin remaining actions" issue.

- **Author is `grtninja`,** likely a security-focused contributor doing a sweep across multiple OSS repos. This pattern (one-PR-per-workflow-file, pin third-party actions, leave version comments) is the signature of someone running OpenSSF Scorecard or similar tooling and submitting upstream fixes. Welcome contribution shape.

## Risk

Effectively zero. The only behavior change is "the workflow will now use exactly these two commits forever, until someone explicitly bumps them." If those commits ever get yanked (extremely unlikely for `actions/checkout` / `actions/setup-python`, both of which GitHub will preserve indefinitely), the workflow would fail loudly — not silently misbehave. That's strictly safer than the current floating-tag setup.

## Verdict

**merge-as-is** — exactly the right shape for an action-pinning PR. Two-line diff, comment-preserved version intent, no stealth version bumps, first-party actions but the principle is universal. Maintainer should land it and consider opening a follow-up to pin any remaining unpinned actions in `.github/workflows/`.

## What I learned

The `# vX.Y.Z` trailing comment after a SHA-pinned action isn't decoration — it's the contract Dependabot reads to propose future bumps in a human-meaningful diff. Without the comment, every dependabot update PR would look like an opaque SHA-to-SHA change. With it, reviewers see "v3.6.0 → v3.7.0" in the PR title/body. The 7-extra-characters-per-line cost buys back massive review-friction savings on every future bump. Worth adopting as a hard rule for any workflow file that pins actions.
