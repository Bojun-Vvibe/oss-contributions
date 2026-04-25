# All-Hands-AI/OpenHands PR #14128 — Support pagination for branch search

- **URL:** https://github.com/All-Hands-AI/OpenHands/pull/14128
- **Head SHA:** `f93bf1cb3e9c3a48a9905f3520cb129405fc1279`
- **Files touched:** 15 (router + 6 provider services + 5 unit tests)
- **Verdict:** `request-changes`

## Summary

Closes #13883: `/git/branches/search` previously rejected `page_id` when
a search query was set with a 400. PR removes that guard and threads
`page` into `search_branches` for GitHub, GitLab, Bitbucket Cloud,
Bitbucket DC, Forgejo, and Azure DevOps.

## Specific references

- `openhands/app_server/git/git_router.py:242-249` — drops the
  "pagination not yet supported" `HTTPException` and forwards `page` to
  the client. Correct router-level change.
- `openhands/integrations/azure_devops/service/branches.py:138-191` —
  Azure DevOps implementation pages by *post-filter* slicing:
  `filtered_branches[start:end]` with `start = max((page-1)*per_page, 0)`.
  This is where my concern is. The previous code broke out of the
  filter loop as soon as `len(filtered_branches) >= per_page`, which
  was at least bounded; the new code iterates the *entire* branches
  list (no upstream paging) and slices afterward. For a repo with
  thousands of branches matching the query, page=1 and page=50 both pay
  the full O(N) scan.
- Same pattern in
  `openhands/integrations/bitbucket/service/branches.py` and
  `forgejo/service/branches.py` (per PR diff metadata; +2/-1 and +4/-2).
- `openhands/integrations/github/queries.py:+6` and
  `github/service/branches_prs.py:+17` — GitHub uses GraphQL cursor
  paging, which is the right model. That part looks fine.
- `tests/unit/app_server/test_git_router.py:+43` and per-provider
  branch tests (+52 for GitHub, +1 each for the others) — router
  pagination is well covered, but the *non-GitHub* provider tests only
  add a single line each. They likely don't exercise the
  large-result-set path where the post-filter slice matters.

## Reasoning / requested changes

1. **Azure DevOps / Bitbucket / Forgejo paging is misleading.** Slicing
   after a full filter pass is not real pagination — it just hides the
   cost. For a stable contract the providers either need to push the
   query into the upstream API, or the response should include
   `has_more` derived from the unsliced length so callers know when
   they've exhausted results. Today, page=N+1 returns `[]` even if
   there were more matches than fit on page N (because the per-page+1
   trick isn't applied to the post-slice). Confirm and fix.
2. **No regression test** for the case "search returns more than
   per_page results, page=2 returns the next slice" on the
   non-GitHub providers. Please add one for at least Azure DevOps,
   since the implementation diverges most.
3. **`per_page=limit + 1` in the router** is the standard
   "is-there-a-next-page" trick. Make sure the post-slice providers
   honour it: with `start = (page-1)*limit`, slicing
   `[start : start + limit + 1]` is what you want. Spot-check this
   against the diff before merging.

Request changes pending (1) and (2).
