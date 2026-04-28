# QwenLM/qwen-code#3690 — fix(ci): use squash merge for SDK release auto-merge

- **Repo:** [QwenLM/qwen-code](https://github.com/QwenLM/qwen-code)
- **PR:** [#3690](https://github.com/QwenLM/qwen-code/pull/3690)
- **Head SHA:** `cb669eeeca60363bc947e17abafe1d0cc4c8065e`
- **Size:** +1 / -1 across 1 file (`.github/workflows/release-sdk.yml`)
- **State:** MERGED

## Context

The Release SDK workflow's `Enable auto-merge for release PR` step was
calling `gh pr merge "${PR_URL}" --merge --auto --delete-branch`, but
the repository only allows squash merges. The PR body verifies via
`gh repo view`:

```
{"mergeCommitAllowed":false,"rebaseMergeAllowed":false,"squashMergeAllowed":true}
```

So the auto-merge step always exited 1 with a "merge method not
allowed" error from `gh`, marking every release run red even when
publish, tag, and release-PR creation all succeeded. The most recent
example: `@qwen-code/sdk@0.1.7` shipped to npm `latest` cleanly, the
release-PR (#3688) was created, but the workflow run
[25036315088](https://github.com/QwenLM/qwen-code/actions/runs/25036315088)
went red at this step and #3688 was left needing manual merge.

## Design analysis

One-character semantic change at `.github/workflows/release-sdk.yml:408`:

```diff
-          gh pr merge "${PR_URL}" --merge --auto --delete-branch
+          gh pr merge "${PR_URL}" --squash --auto --delete-branch
```

The `--merge`/`--squash`/`--rebase` flag tells `gh pr merge` which
of GitHub's three merge methods to use. The repo-level "allowed merge
methods" setting at `https://github.com/QwenLM/qwen-code/settings`
gates which of those the API will accept. The flag must be in the
intersection of (locally requested, repo-allowed) for the API call
to succeed.

The PR body shows the right diagnostic chain:
1. Symptom: workflow always red at this step
2. Hypothesis: merge method mismatch
3. Verification: `gh repo view --json mergeCommitAllowed,
   squashMergeAllowed,rebaseMergeAllowed` showing only
   `squashMergeAllowed: true`
4. Fix: align flag with the only allowed method

The `--auto` flag is preserved (still queues for auto-merge once
required checks pass) and `--delete-branch` is preserved (still
deletes the source branch on merge). Only the merge-method
selector changes.

## Risks / nits

1. **No corresponding workflow-side defense against future repo-policy
   drift.** If the repo settings ever flip again (e.g., admin enables
   merge commits and disables squash), this fix breaks the same way.
   Worth either:
   - A pre-step that calls `gh repo view --json squashMergeAllowed`
     and warns/fails fast if squash isn't allowed, or
   - A small runtime fallback that tries `--squash` first and falls
     back to `--merge` if the API rejects with the specific
     "method not allowed" error.

   Probably overkill for a small repo, but worth at least a comment
   in the YAML explaining *why* `--squash` is the right choice
   (linking back to the repo settings) so the next maintainer
   understands the coupling.

2. **The Test Plan checkboxes are unchecked.** The PR body's "Test
   plan" lists two boxes (next stable SDK release run completes
   green; release PR auto-merges via squash with branch deleted) but
   neither is checked. That's expected for a pre-merge state of this
   workflow fix — the verification can only happen on the next
   release run. Worth a follow-up comment after the next release
   confirming both checks passed, so the fix has a recorded
   verification.

3. **`--delete-branch` semantics with `--auto`.** When using
   `--auto`, GitHub queues the merge to fire when required checks
   pass; the `--delete-branch` flag is honored at *that* time, not
   immediately. So if the auto-merge never fires (checks failed,
   PR closed manually), the branch persists. This is GitHub's
   semantics and not something the PR can fix, but worth a comment
   in the YAML.

4. **The PR doesn't mention whether #3688 was merged after this
   fix lands.** The PR body says "PR #3688 was left needing
   manual merge" — the implicit follow-up is "this fix lets the
   *next* release auto-merge, but #3688 still needs human action."
   Worth either explicitly merging #3688 manually as part of this
   PR's flow, or a comment confirming someone did.

5. **The "Create Issue on Failure" step at `:411` is not visible
   in the diff but is referenced.** That step presumably fires
   when the auto-merge step exits non-zero — meaning every release
   run before this fix produced a tracking issue. Worth checking
   whether those issues exist, are now stale, and need cleanup.

## Verdict

**merge-as-is.** The fix is correctly minimal (one character in one
line), the diagnosis chain in the PR body is well-shown, and the
verification command (`gh repo view --json ...`) is reproducible.
Nothing about the change should block merge. Nits are all
post-merge follow-ups, none structural.

## What I learned

- The intersection of (locally-requested merge method, repo-allowed
  merge methods) must be non-empty for `gh pr merge --auto` to
  succeed. Repos with restrictive merge-method policies are a
  recurring footgun for auto-merge workflows that hardcode
  `--merge`.
- `gh repo view --json mergeCommitAllowed,squashMergeAllowed,
  rebaseMergeAllowed` is the right diagnostic command for this
  class of CI failure — it prints the repo's allowed-methods
  state directly so the mismatch is obvious. Worth adding to a
  CI-debugging cheat sheet.
- One-character workflow fixes that turn every release run from
  red to green are high-value: the failed-step noise was
  presumably masking *real* failures and producing alert fatigue,
  so the fix's payoff is more than just the literal one character.
- Pre-step "policy verification" (checking that the repo state
  the workflow assumes is actually true) is a useful pattern for
  workflows that hardcode environment assumptions. Cheap to add,
  catches drift before it produces confusing failures downstream.
