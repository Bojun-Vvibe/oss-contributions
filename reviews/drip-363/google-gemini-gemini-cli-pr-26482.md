# google-gemini/gemini-cli #26482 — CI Optimization & Lifecycle Manager Hardening

- Head SHA: `f9840e7efaa6674b89b5f8ed7d8ad13ab7ac44f9`
- Author: gemini-cli-robot (self-modifying bot PR)
- Diff: +498 / −222 across 9 files (CI workflow, lifecycle script, brain workflow, brain prompts, plus one self-deletion of the now-redundant Critique-as-editor flow)

## Verdict: `needs-discussion`

## What it does

Three loosely-related changes bundled into one PR:

### A. CI matrix syntax fix + cost reduction (`.github/workflows/ci.yml:147-156, 239-248`)

- Replaces a literal-string node-version list with `${{ fromJSON(github.event_name == 'pull_request' && '["20.x"]' || '["20.x", "22.x", "24.x"]') }}` so PRs run only on Node 20.x while `main`/`release/**` keep the full triple.
- Reorders `npm ci` to run **before** `npm run build` in two jobs (Linux + Mac test). Previously build ran first and could fail if the build script touched any locally-installed tool.

### B. Lifecycle manager scaling + grace-period rewrite (`.github/scripts/gemini-lifecycle-manager.cjs`)

- `processItems` switched from a single `search.issuesAndPullRequests` call (capped at 100 items) to `github.paginate(...)` so the bot can process the full backlog (PR claims >2000 issues).
- Removed the outer `try/catch` around the search call and made the per-item `try/catch` the only error boundary (each item's failure is logged but doesn't poison the rest).
- `status/need-information` removal query gets `comments:>1` appended to skip issues with no contributor response yet.
- PR nudge logic is rewritten:
  - Old close query: `created:<${prCloseThreshold}` — closed any PR over 14d old that lacked `help wanted`, regardless of whether it had been nudged.
  - New close query: `label:"${PR_NUDGE_LABEL}" -label:"help wanted" ... created:<${prCloseThreshold} updated:<${prNudgeThreshold}` — requires the nudge label to be present AND no activity in the last 7 days *after* the nudge.
- Old nudge query: `created:${prCloseThreshold}..${nudgeThreshold}` — narrow window (between 7d and 14d old). New: `created:<${prNudgeThreshold}` — every un-nudged PR over 7d gets a nudge.

### C. Brain → Critique becomes a `MAX_ITERATIONS=4` self-corrective loop (`.github/workflows/gemini-cli-bot-brain.yml:120-238`) + Critique demoted from editor to evaluator (`tools/gemini-cli-bot/brain/critique.md`)

- The previously-separate "Run Brain Phases" and "Run Critique Phase" steps are merged into a single bash loop: Brain runs, Critique evaluates, on `[REJECTED]` the loop captures `critique_output.log` into `critique_feedback.md`, calls `git reset && git checkout .`, deletes scratch files, and re-runs Brain with the feedback prepended. Up to 4 iterations.
- Critique's prompt is rewritten to forbid editing: *"You are an evaluator ONLY. You MUST NOT apply fixes or modify the code yourself."* — prior version explicitly said *"you MUST directly edit the scripts to fix the issue and stage the fixes using `git add <file>`"*.
- New `Defensive Scripting & Resilience (MANDATORY)` section in `brain/common.md:91-105` requires per-item try/catch and explicit preservation of label exemptions.
- New PR `REF` line in the Critique-result-fetcher step: `REF="${{ github.ref }}"` (was hardcoded `REF="main"`).

## Why I'm holding this for discussion

### 1. Self-modifying-bot PR + 4× API cost multiplier without quantification

The same bot author (`gemini-cli-robot`) is proposing changes to its own brain/critique loop that quadruples per-trigger API calls (1 Brain + 1 Critique → 4× (Brain + Critique) worst case = 8 model calls per workflow run, vs. previous 2). The PR text frames this as "Backlog Management" and "Contributor Experience" but does not quantify:

- What's the daily trigger volume? `issue_comment` + `pull_request_review_comment` + manual dispatch on a repo with >2000 open issues could be hundreds of triggers/day.
- What's the per-trigger token cost on `gemini-3-flash-preview`? Worst case is now 4× the prior cost.
- What's the GitHub API rate-limit budget impact? `github.paginate` for issue search is 1 RPC per 100 items × N pages × multiple `processItems` calls per run, vs. previously a single capped call.

Self-modifying-bot PRs that expand bot autonomy (more iterations, more API calls, broader paginate scope) are exactly the case where the original author cannot be the merger and the cost/rate-limit math needs explicit maintainer sign-off.

### 2. Three orthogonal changes bundled — bisect surface is wrong

The CI matrix fix (~5 lines, low risk), the lifecycle manager rewrite (~50 lines, medium risk to OSS contributors via PR nudge/close policy), and the Brain/Critique self-corrective loop (~120 lines, high risk to API budget + bot autonomy) are independent and should be three separate PRs. If the loop turns out to thrash and needs reverting, the CI matrix fix and the (genuinely good) `comments:>1` triage optimization will be reverted along with it.

### 3. The new "discard rejected changes" path is silent

`.github/workflows/gemini-cli-bot-brain.yml:217-224` does:
```bash
git reset
git checkout .
rm -f pr-description.md branch-name.txt pr-comment.md pr-number.txt issue-comment.md bot-changes.patch rejected-changes.patch
```

This nukes whatever the Brain produced, with no `git diff > rejected-iteration-N.patch` capture per iteration. Only the final iteration's `rejected-changes.patch` is uploaded. Debugging "why did iterations 1-3 all get rejected" requires re-reading the action logs and inferring from the critique output, which is awkward — capturing a per-iteration patch artifact (`rejected-iteration-${ITERATION}.patch`) would cost almost nothing and would make iteration-loop diagnosis tractable.

### 4. `REF="${{ github.ref }}"` change at line 282 is buried in this PR but materially affects which branch the post-Critique fetch operates against

The previous code hardcoded `REF="main"`. The new code uses `${{ github.ref }}`, which in PR triggers can be `refs/pull/<n>/merge`. This is probably the right behavior — the bot should fetch from the PR's branch, not main — but it's a meaningful semantic change that deserves its own PR, its own test, and an explicit explanation. Buried in a 498-line PR titled "CI Optimization & Lifecycle Manager Hardening", it's easy for a maintainer to miss.

### 5. Lifecycle close-policy change — are existing nudged PRs grandfathered?

The new close query is `label:"${PR_NUDGE_LABEL}" ... created:<${prCloseThreshold} updated:<${prNudgeThreshold}` — requires both the nudge label AND no update for 7d after the nudge. The old query closed any 14d-old PR. **Are there existing PRs that were nudged under the old policy** (so they have `status/pr-nudge-sent`) and have been sitting untouched? Under the new policy, those will be closed *immediately* on the next run because both conditions trivially hold. The PR doesn't address the migration boundary — a one-time `--dry-run` of the new close query against the existing labelled-PR set should be in the validation, otherwise the rollout could close a batch of OSS PRs in a single sweep with no warning.

### 6. The `try/catch` removal in `processItems` is a regression for unauthenticated/rate-limited search

Old code:
```js
try {
  const response = await github.rest.search.issuesAndPullRequests({...});
  ...
} catch (err) {
  core.error(`Search failed: ${err.message}`);
}
```

New code:
```js
const items = await github.paginate(github.rest.search.issuesAndPullRequests, {...});
core.info(`Found ${items.length} items.`);
for (const item of items) { ... }
```

There's no try/catch around `github.paginate`. If the search fails (rate limit, transient 502), the entire `lifecycle-manager` run aborts at the first failed `processItems(...)` call instead of logging and moving on to the next category. Combined with the new pagination exhausting more API budget, the failure mode is "first transient blip kills the whole run" instead of "log and continue". Restoring the outer try/catch on the paginate call is a one-line fix.

## What's actually good in here (would land in a focused split)

- The `comments:>1` filter on the `status/need-information` removal query (`gemini-lifecycle-manager.cjs:62`) is a clear N+1 optimization with no behavioral change.
- `npm ci` before `npm run build` (`ci.yml:164, 256`) is unambiguously correct.
- Critique demoted to evaluator (`brain/critique.md`) is a good separation-of-concerns change *if* the iteration loop is sound — but it's coupled to the loop here.
- New `Defensive Scripting & Resilience` section in `brain/common.md:91-105` codifies real lessons (per-item try/catch, preserve exemptions). The instruction *"Never drop existing protections or safety checks unless you have proven they are the explicit root cause of the issue"* is exactly the guardrail that would have prevented prior incidents in this same script.

## Recommended path forward

Split into three PRs:
- PR-A: `ci.yml` matrix fix + step reorder + `npm ci`/`build` swap. Trivially mergeable.
- PR-B: lifecycle-manager pagination + `comments:>1` triage optimization + try/catch restoration. Behavioral change to OSS-PR close policy gets its own validation cycle (dry-run against existing nudged set, grandfathering plan).
- PR-C: Brain/Critique self-corrective loop + Critique-as-evaluator + `common.md` `Defensive Scripting` section + `${{ github.ref }}` fetch fix. This one needs explicit cost/rate-limit math, per-iteration artifact capture, and maintainer sign-off because it expands bot autonomy.

## Citations

- `.github/workflows/ci.yml:147-156, 239-248` — node-version sharding via `fromJSON`
- `.github/workflows/ci.yml:164-167, 252-258` — `npm ci`/build reorder
- `.github/scripts/gemini-lifecycle-manager.cjs:36-58` — pagination switch + try/catch removal
- `.github/scripts/gemini-lifecycle-manager.cjs:62` — `comments:>1` triage filter
- `.github/scripts/gemini-lifecycle-manager.cjs:163-200` — PR nudge/close policy rewrite
- `.github/workflows/gemini-cli-bot-brain.yml:122-238` — `MAX_ITERATIONS=4` self-corrective loop
- `.github/workflows/gemini-cli-bot-brain.yml:282` — `REF="main"` → `REF="${{ github.ref }}"`
- `tools/gemini-cli-bot/brain/critique.md:1-30` — Critique demoted from editor to evaluator
- `tools/gemini-cli-bot/brain/common.md:91-105` — new Defensive Scripting section
