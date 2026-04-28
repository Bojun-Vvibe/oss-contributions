# QwenLM/qwen-code #3703 — fix(ci): use squash auto-merge for sdk release pr

- PR: https://github.com/QwenLM/qwen-code/pull/3703
- Author: doudouOUC (jinye)
- Head SHA: `22fed3ae4d8a`
- Base: `main` · State: OPEN · Fixes #3689
- Diff: 1+/1- across 1 file (`.github/workflows/release-sdk.yml`)

## Verdict: merge-as-is

## Rationale

- **Right one-character fix for a workflow that couldn't have been working.** At `release-sdk.yml:408`, `gh pr merge "${PR_URL}" --merge --auto --delete-branch` becomes `gh pr merge "${PR_URL}" --squash --auto --delete-branch`. The PR body cites the exact failure: `gh run view 25036315088 --repo QwenLM/qwen-code --log-failed` returned `GraphQL: Merge method merge commits are not allowed on this repository`. The repo settings (per `gh repo view ... --json mergeCommitAllowed,...`) are `mergeCommitAllowed: false`, `rebaseMergeAllowed: false`, `squashMergeAllowed: true`, `viewerDefaultMergeMethod: SQUASH`. The workflow was using a merge method the repo *categorically* rejects — so this fix is unblocking, not a behavior change.
- **The repo policy is itself the strongest evidence.** Both `mergeCommitAllowed: false` *and* `rebaseMergeAllowed: false` means squash is the only legal merge method; the workflow had no other valid option. Picking `--squash` matches `viewerDefaultMergeMethod: SQUASH` which is the policy the repo's own UI defaults to. Zero ambiguity in the choice.
- **Auto-merge semantics survive the swap.** `--auto` waits for required checks before merging; `--delete-branch` cleans up after. Both flags work identically with `--squash` and `--merge` — only the merge method changes. SDK release PRs are by-definition single-author single-purpose ("bump version to v0.1.7"), so squash-collapse of any fixup commits is the *right* shape anyway: the resulting `main` history gets a single clean release commit per version, not a release commit + fixups. That's the standard release-PR convention regardless of repo policy.
- **PR body cites the exact failed run and the canonical evidence command.** `gh run view 25036315088 --repo QwenLM/qwen-code --log-failed` is the right anchor. The `gh repo view` command output establishes the repo policy as load-bearing context. Reviewer can verify in 30 seconds.

## Nits / follow-ups

- **The non-SDK release workflow `release.yml` should be checked for the same bug.** The PR only patches `release-sdk.yml`, but the SDK workflow's structure (auto-merge of a release PR via `gh pr merge --merge`) is the kind of pattern that gets copy-pasted across release workflows. A quick `rg --no-heading "gh pr merge.*--merge" .github/workflows/` would confirm whether the same one-character fix is needed elsewhere. If yes, a single follow-up commit; if no, a comment in the PR confirming.
- **Worth a CI lint rule that forbids `gh pr merge --merge` in workflows when `mergeCommitAllowed: false`** — the same way Renovate/Dependabot do compatibility checks. But that's tooling-side, not in scope for this PR.
- **The commit message could note the repo-policy mismatch as the cause** ("squashMergeAllowed: true is the only enabled method") rather than just "use squash auto-merge" — saves the next person the same `gh repo view` investigation.

## What I learned

CI workflows that call `gh pr merge --merge` on repos that have `mergeCommitAllowed: false` are dead code paths the *first time they're triggered* — they look correct in review (the syntax is fine, the auto-merge contract is fine, the delete-branch flag is fine), but GitHub rejects them at the GraphQL layer with a cryptic error that doesn't mention "your workflow is using the wrong merge method." The structural fix is to detect the mismatch at workflow-author time, not at first-failure time: a pre-merge CI check that runs `gh repo view --json ...` and asserts the workflow's `--merge|--squash|--rebase` flag matches an *enabled* method would catch this before the release PR is queued. Until that exists, the diagnostic discipline is "any GraphQL `Merge method ... not allowed` error → check repo policy → swap the method." This PR demonstrates exactly that diagnostic chain in the PR body — which is the high-quality form for a one-character fix.
