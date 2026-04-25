# All-Hands-AI/OpenHands #14132 — fix: support pagination for branch search

- **PR:** https://github.com/All-Hands-AI/OpenHands/pull/14132
- **Head SHA:** `f2473c7b16cd8911b83c968bbc1cfc20e6a9a40c`
- **Files changed:** ~10 across `git_router.py` and per-provider `branches.py` files (Azure DevOps, Bitbucket Cloud, Bitbucket Data Center, Forgejo, plus presumably GitLab/GitHub/Gitea given the diff size of ~933 lines)

## Summary

Removes the longstanding 400-error on `?query=...&page=>1` in the branches search endpoint by collapsing the historically-bifurcated "search vs. list" code paths into a single paginated `get_branches(query=...)` call across every git-provider integration. The router (`git_router.py:241-256`) previously had a `TODO(#13883)` explicit `HTTPException` for this case; this PR is the resolution of that TODO.

## Line-level call-outs

- `openhands/app_server/git/git_router.py:241-256` — old code did `if query: search_branches(per_page=limit+1)` (no pagination) `else: get_branches(page, per_page=limit+1)`. New code is one branch: `client.get_branches(repository, page, per_page=limit, query=query or None)` followed by `current_page.has_next_page` check. Cleaner; the `+1` over-fetch trick is dropped because the providers now report `has_next_page` directly. Verify every provider's `get_paginated_branches` actually returns a populated `has_next_page` — if any returns `False` constantly, paging silently breaks.
- `openhands/integrations/azure_devops/service/branches.py:67-105` — `get_paginated_branches` now accepts `query: str | None`. Filtering happens **client-side** at `:101-108` after the API call: `branch_data = [b for b in branch_data if lowered_query in b['name']...]`. **This is the same correctness gap as before, just relocated**: pagination is computed *after* filtering, but the API returned an unpaginated list, so `page=N` against a sparse query may return inconsistent counts depending on the underlying API page size. ADO's `/refs?filter=heads/` doesn't natively support name-substring search, so this client-side filter is the only option, but please document the constraint in the docstring (`page` here means "page of the filtered subset", not "page of the API result").
- `openhands/integrations/azure_devops/service/branches.py:147-159` — `search_branches` is now a thin shim over `get_paginated_branches(query=query)`. Good consolidation. Returns `[]` on `not query` early — preserves prior contract.
- `openhands/integrations/bitbucket/service/branches.py:64-66` — adds `params['q'] = f'name~"{query}"'` for Bitbucket Cloud. This **uses the API's native filter** (BBQL `name~"..."`), so pagination is server-side and correct. Note: the query value is interpolated into a quoted BBQL expression — if `query` contains `"`, it breaks the BBQL parse. Suggest a small escape (`query.replace('"', '\\"')`) or rejecting queries with `"` at the router.
- `openhands/integrations/bitbucket_data_center/service/branches.py:59-60` — uses `params['filterText'] = query`. Native server-side filter; also pagination-safe.
- `openhands/integrations/forgejo/service/branches.py:298-305` (visible in tail not shown above) — appears to fetch `get_branches(repository)` and filter client-side via `lowered = query.lower()`. Same caveat as ADO: pagination is over the filtered set, not the API set. If forgejo branch counts can be very large, this is O(n) per request.
- `openhands/app_server/git/git_router.py:251` — `query=query or None` correctly normalizes empty-string to `None`. Good.
- `openhands/app_server/git/git_router.py:255` — `if current_page.has_next_page: next_page_id = encode_page_id(page + 1)`. Replaces the `len(branches) > limit` check. Correct, assuming providers honestly report `has_next_page`.
- Removed import: `Branch` from `service_types` is no longer needed in the router (`:30`). Clean.
- **Tests:** the diff text shown does not include any test files — for a 933-line diff touching every provider's branch logic, that's a substantial gap. At minimum, one test per provider showing `?query=foo&page=2` returns the expected items would catch the most likely regression (off-by-one in client-side slicing for ADO/Forgejo).
- **Behavioural regression risk:** prior `search_branches` for ADO fetched commit-date metadata for each filtered branch (the `commit_url` per-branch lookup in the old code at the deleted `:122-138` block). The new path goes through `get_paginated_branches`, which (per the visible diff) does not do the per-branch commit fetch. **This means `last_push_date` may now be `None` on ADO search results when it previously was populated.** Worth confirming — if true, downstream UI sorting by date may break silently.

## Verdict

**request-changes**

## Rationale

The consolidation is the right architectural call and resolves a long-standing TODO (#13883). Two concrete blockers: (1) the **ADO `last_push_date` regression** — the old `search_branches` did a per-branch commit lookup; the new path doesn't appear to. If users sort search results by recent activity, this silently degrades. Either restore the lookup or explicitly document that search results lack commit dates. (2) **No tests** for a refactor that touches ~5 provider implementations. A minimal "search + page=2 returns expected slice" matrix per provider is essential. Two nice-to-haves: BBQL quote-escaping in Bitbucket Cloud, and a docstring note on ADO/Forgejo that pagination is over the filtered subset. Once the regression is addressed and a few tests land, this is merge-after-nits.
