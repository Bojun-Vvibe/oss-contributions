# All-Hands-AI/OpenHands #14099 — fix(git): support pagination for branch search queries

- **Repo**: All-Hands-AI/OpenHands
- **PR**: [#14099](https://github.com/All-Hands-AI/OpenHands/pull/14099)
- **Head SHA**: `fb98b3c9c2b52077588d590a7105464859ba6b71`
- **Author**: kayametehan
- **State**: OPEN (+268 / -46)
- **Verdict**: `merge-after-nits`

## Context

The branch-search router in `git_router.py` previously
hard-blocked any `page != 1` request with a 400 ("Pagination
not yet supported for branch search queries. Use empty query
to list all branches with pagination.") with a TODO referencing
issue #13883. This PR removes that block and threads `page`
through every provider's `search_branches` implementation
(GitHub, GitLab, Bitbucket Cloud, Bitbucket Data Center,
Azure DevOps, Forgejo) plus the abstract `service_types.py`
interface.

## Design

Six provider-specific implementations + one router edit + one
interface edit + tests.

1. **Router unblock** (`openhands/app_server/git/git_router.py:242-250`).
   The 400 short-circuit goes away; `page=page` is added to
   the `client.search_branches` call alongside the existing
   `per_page=limit + 1` (the `+1` is the standard "look-ahead
   to compute `has_more`" trick).

2. **Interface widening** (`openhands/integrations/service_types.py:529`)
   adds `page: int = 1` to the abstract
   `search_branches` signature. Same change ripples through
   `provider.py:330-355` so the dispatcher forwards `page=` as
   a kwarg (good — kwarg-by-name protects against future
   reorderings of the signature).

3. **GitHub (GraphQL)** (`integrations/github/queries.py:131-145`,
   `integrations/github/service/branches_prs.py:117-160`).
   This is the most substantial change. GraphQL doesn't have
   numeric pagination — it has cursor-based `after: $cursor`
   pagination via `pageInfo { hasNextPage endCursor }`. The
   PR walks pages 1..N in a loop, advancing `cursor` from the
   previous page's `endCursor`, and only returns the nodes
   from the **target** page. Correct shape, but **O(page)**
   cost: requesting page 50 means 50 round-trips. For a
   user-facing branch-search dropdown that's almost always
   pages 1-3, that's fine, but a malicious or buggy client
   asking for page 1000 would issue 1000 GraphQL requests.
   Worth a `page = min(page, 100)` clamp similar to the
   `per_page = min(max(per_page, 1), 100)` already on
   line 117.

4. **GitLab** (`integrations/gitlab/service/branches.py:89-95`)
   — GitLab supports numeric pagination natively, so the fix
   is a one-line `'page': str(page)` in the params dict. Clean.

5. **Bitbucket Cloud** (`integrations/bitbucket/service/branches.py:84-100`)
   — same shape, `'page': page` in the `pagelen`-keyed params.
   Clean.

6. **Bitbucket Data Center**
   (`integrations/bitbucket_data_center/service/branches.py:69-100`)
   — uses `start = max((page - 1) * per_page, 0)` and passes
   it as the `start` query param. Correct mapping from
   page-based to offset-based pagination.

7. **Azure DevOps** (`integrations/azure_devops/service/branches.py:135-188`)
   — fetches all branches and slices in Python:
   `start_idx = max((page - 1) * per_page, 0); end_idx =
   start_idx + per_page; return filtered_branches[start_idx:end_idx]`.
   Note the inner `if len(filtered_branches) >= end_idx: break`
   inside the filter loop is a real optimisation (stop
   accumulating once we've passed the requested page's end),
   but it relies on the outer service returning enough raw
   `branches_data` to fill `end_idx` filtered results — for
   a search query that matches sparsely on a large repo, this
   degrades to "fetch everything, return page N". Acceptable
   for a first pass.

8. **Forgejo** (`integrations/forgejo/service/branches.py:65-72`)
   — same client-side slice as Azure DevOps but without the
   short-circuit (Forgejo doesn't have a server-side filter
   for this endpoint, so the loop fetches all branches first
   anyway).

## Risks

- **GitHub GraphQL O(page) cost** is the biggest concern.
  Adding a `page = min(max(page, 1), 100)` clamp at
  `branches_prs.py:118` (next to the existing `per_page`
  clamp) prevents adversarial inputs from issuing thousands
  of GraphQL requests in a single call. The router's
  `decoded_page_id` decoding path should also probably enforce
  a sane upper bound — happy to merge first, follow up second.
- **Azure DevOps page-fill assumption.** The
  `if len(filtered_branches) >= end_idx: break` plus
  `return filtered_branches[start_idx:end_idx]` works only if
  the upstream `branches_data` list contains enough entries
  for `end_idx` filtered results. If the upstream API is
  itself paginated and the PR doesn't iterate over its pages,
  later pages will silently return empty even when more
  matches exist. Worth adding a comment + a TODO if the
  upstream pagination isn't being threaded through here.
- **Forgejo client-side slice** has no upper bound on the
  raw `get_branches(repository)` call — for a repo with
  10k branches that's a 10k-element fetch on every search
  page request. A `per_page * page` early-stop bound, or
  switching to Forgejo's native `?page=` if that endpoint
  supports it, would be a real follow-up.
- **Test coverage** is across `test_git_router.py` plus
  per-provider branch tests — the diff shows
  `test_azure_devops_branches.py`, `test_bitbucket_branches.py`,
  `test_bitbucket_dc_branches.py`, `test_forgejo_branches.py`,
  and `test_github_branches.py` all touched, which is
  appropriate fan-out for a change that touches every
  provider. Worth confirming each new test asserts both
  `page=1` (regression) and `page=2` (new path).

## Suggestions

- Add `page = min(max(page, 1), 100)` to the GitHub branch
  in `branches_prs.py:118` to bound the GraphQL round-trip
  cost.
- Add a TODO + comment on the Azure DevOps slice noting the
  page-fill assumption (and that upstream API pagination
  isn't being walked).
- Consider a follow-up to use Forgejo's native pagination if
  available, rather than the client-side slice.
- Confirm each new per-provider test exercises both `page=1`
  (default) and `page=2+` (new behaviour).

## Verdict reasoning

`merge-after-nits`. The fix is the right shape (unblock the
router, push pagination down into each provider with the
provider's native idiom) and the test fan-out is good. The
GitHub O(page) cost is a real issue that should be addressed
in this PR — a one-line clamp — before landing. Other items
are follow-ups.

## What I learned

Page-based pagination on top of cursor-based backends is
always O(page) unless you cache cursors keyed by `(query,
per_page, page)`. For a UI search-dropdown context where
the user is almost never past page 3, the trade-off is
"accept O(page) up to a clamp and let 99% of requests stay
fast." But the clamp is non-optional — without it, a single
client requesting `?page=999` is a 999-RTT amplification
attack against your GraphQL quota. Same shape as the
classic "OFFSET in SQL is O(offset)" problem; same answer:
clamp the upper bound and recommend keyset pagination
elsewhere if users actually need to deep-page. This PR's
shape is right, just one clamp short.
