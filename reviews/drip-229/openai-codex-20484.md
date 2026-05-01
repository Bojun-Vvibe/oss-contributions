# openai/codex #20484 — [codex] Improve PR babysitter CI diagnostics and guardrails

- **PR**: https://github.com/openai/codex/pull/20484
- **Head SHA**: `d59bc8ad420c`
- **Files reviewed**: `.codex/skills/babysit-pr/SKILL.md`, `.codex/skills/babysit-pr/agents/openai.yaml`, `.codex/skills/babysit-pr/references/github-api-notes.md`, `.codex/skills/babysit-pr/references/heuristics.md`, `.codex/skills/babysit-pr/scripts/gh_pr_watch.py`, `.codex/skills/babysit-pr/scripts/test_gh_pr_watch.py`
- **Date**: 2026-05-01 (drip-229)

## Context

Two-axis hardening of the PR-babysitter skill that lives inside the
codex repo (skill-as-code; the agent reads its own `SKILL.md` at
runtime). Axis one is **diagnosis latency**: today the watcher waits
for the overall workflow `run` to terminate before `gh run view
--log-failed` returns useful logs, which can leave Codex idling for
minutes after a single fast-failing job has already produced a
diagnosable trace. Axis two is a **safety guardrail**: prior versions
allowed the agent to (a) post replies to human-authored review comments
without confirmation and (b) "fix" CI by patching unrelated flaky
tests, runner-config, dependency pins, etc. The PR closes both holes.

## Diff (6 files, +164 -14)

### Watcher script (`scripts/gh_pr_watch.py`)

New `get_jobs_for_run()` and `failed_jobs_from_workflow_runs()` at
`gh_pr_watch.py:341-396` query the per-run jobs API
(`repos/{owner}/{repo}/actions/runs/{run_id}/jobs`) and emit a
`failed_jobs[]` entry for every job whose `conclusion` is in
`FAILED_RUN_CONCLUSIONS`, regardless of whether the parent run is still
`in_progress`. Each failed-job entry carries the direct
`logs_endpoint = "repos/{owner}/{repo}/actions/jobs/{job_id}/logs"`
that the agent can fetch immediately.

`recommend_actions(...)` at `:631` extends the
`has_failed_pr_checks` predicate to OR in `bool(failed_jobs)` so the
`diagnose_ci_failure` action surfaces as soon as a single job fails,
not after the whole run completes.

`collect_snapshot()` at `:683-707` plumbs `failed_jobs` through to
the snapshot payload.

### Test (`scripts/test_gh_pr_watch.py`)

`test_failed_jobs_include_direct_logs_endpoint` at `:166-217` is the
load-bearing contract pin. It seeds an `in_progress` run with one
failing job + one passing job, monkeypatches `get_jobs_for_run`, and
asserts the exact `failed_jobs[0]["logs_endpoint"] ==
"repos/openai/codex/actions/jobs/555/logs"` shape — pinning **both**
the early-surfacing behavior (run still `in_progress`) and the URL
construction. Plus three other tests updated to thread the new
`failed_jobs` arg through `recommend_actions` / snapshot fixtures.

### Skill prose

`SKILL.md:30,72-76,99,108,135,140` and `references/heuristics.md:21,30-34,47,64`
add three concrete prose contracts:

1. **No automatic GitHub replies to human-authored review comments.**
   The old `SKILL.md:33` instruction "post one reply on the
   comment/thread" is replaced by "Do not post replies to human-
   authored review comments/threads unless the user explicitly
   confirms the exact response." Mirrored in `agents/openai.yaml:4`
   default prompt.

2. **No "fix" of unrelated flaky/infra failures.** New paragraph at
   `SKILL.md:80` and decision-tree arm at `heuristics.md:32`: "If
   likely flaky/unrelated and not safely rerunnable: stop and report
   the blocker; do not edit unrelated tests, build scripts, CI
   configuration, dependency pins, or infrastructure code."

3. **Prefer direct job-log endpoint.** `SKILL.md:71-76` and
   `github-api-notes.md:25-30` document the new
   `gh api .../actions/jobs/{job_id}/logs` path with `gh run view
   --log-failed` demoted to a fallback "after the overall workflow run
   is complete."

## Observations

1. **Contract pinned by exact URL, not by `expect.any(...)`.** The
   `logs_endpoint` assertion at `test_gh_pr_watch.py:215` is an exact
   string. A future "helpful" path-builder rewrite that produces the
   same logical endpoint via different formatting will fail-loud, which
   is the right pin for a contract that other agent tooling and
   downstream `gh api` invocations must consume verbatim.

2. **Run filter is correct.** `failed_jobs_from_workflow_runs` at
   `:362-364` only skips a run when it is `completed` AND the run
   conclusion is *not* in `FAILED_RUN_CONCLUSIONS` — i.e. it keeps
   `in_progress` runs (so the early-surface path works) and keeps
   completed-failed runs (so existing post-completion behavior still
   works). The `head_sha` filter at `:359-360` correctly scopes to the
   current PR commit so a stale failure on an older SHA doesn't get
   surfaced.

3. **`process_review_comment` is now stricter.** Combined with the
   `SKILL.md:36` rewording, the agent will surface review items but
   refuse to auto-reply. The existing watcher will still emit a
   `process_review_comment` action (no code change needed) and the
   prose change is the only behavioral guard. That's a reasonable
   trust boundary: the deterministic watcher reports facts, the LLM
   plus the prose contract decides not to act on them.

4. **Three prose-only arms have real teeth.** `heuristics.md:64`
   ("A human review comment requires a written GitHub reply instead
   of a code change") becomes a stop-and-ask condition — a future agent
   that "helpfully" replies to a question will violate the explicit
   list, which is auditable.

## Nits

- **No test for the no-auto-reply prose contract.** The watcher tests
  pin the diagnose-CI behavior; nothing pins the "do not post
  replies" rule. A unit test that exercises a path producing a
  `process_review_comment` action and asserts the snapshot does *not*
  contain a `post_reply` action would be the analogous pin (or a
  fixture-comparison test that shows the recommended-actions list is
  unchanged from before this PR for the human-authored case).

- **`get_jobs_for_run` per-run API call is N+1.** For a PR with N
  workflow runs at the same SHA, the new path makes N additional `gh
  api` calls per snapshot. For PRs with very many parallel workflows
  this could slow the watcher. Consider batching or accepting it
  explicitly with a comment naming the tradeoff.

- **`run_status.lower()` at `:362` but `conclusion` not lowered.**
  Asymmetric. Both come from the GitHub API as lowercase strings in
  practice but the asymmetry is worth a one-line comment or normalize
  both.

- **`agents/openai.yaml:4` default_prompt is now ~580 chars.** It
  inlines two negated rules ("Do not post replies..." and "Do not
  patch unrelated flaky tests..."). Consider extracting to bullet
  list or referencing the SKILL.md sections by anchor so the rules
  are maintained in one place.

## Verdict

`merge-after-nits` — well-scoped, the contract pin (`logs_endpoint`
exact-string assertion) is the right shape, the prose rules close real
abuse vectors, and the watcher script changes are minimal and
covered. Nits are an absent test for the "no auto-reply" rule, the
N+1 call pattern, and the asymmetric case-normalization.
