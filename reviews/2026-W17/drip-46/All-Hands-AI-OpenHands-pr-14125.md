# All-Hands-AI/OpenHands #14125 — fix(integrations): guard Bitbucket Data Center against null nested fields

- **PR:** https://github.com/All-Hands-AI/OpenHands/pull/14125
- **Head SHA:** `090a1830979a871984142e43e20ea889ab8fd134`
- **Files changed:** 4 — `openhands/integrations/bitbucket_data_center/service/base.py` (+8/−1), `openhands/integrations/bitbucket_data_center/service/resolver.py` (+10/−2), `tests/unit/integrations/bitbucket_data_center/test_bitbucket_dc_repos.py` (+34), `tests/unit/integrations/bitbucket_data_center/test_bitbucket_dc_resolver.py` (+24).

## Summary

Two crash-on-null sites in the Bitbucket Data Center integration that mirror cloud-Bitbucket fixes already in flight (#14070, #14085). Site 1: `_parse_repository` did `repo.get('project', {}).get('key', '')` and crashes with `AttributeError: 'NoneType' object has no attribute 'get'` when DC returns `"project": null` (orphaned/admin-listed repos and proxy-stripped payloads). Site 2: `_process_raw_comments` did `comment_data.get('author', {}).get('slug')` for the same anti-pattern reason — the `.get('author', {})` only short-circuits when the key is *absent*, but DC sends the key with a *null value* for deleted/anonymous authors, blowing up the entire comment-ingestion loop.

## Line-level call-outs

- `service/base.py:234-239` — replacement is the textbook idiom: `project = repo.get('project') or {}; project_key = project.get('key', '') if isinstance(project, dict) else ''`. The `isinstance` guard handles the "non-dict, non-None" case (e.g. `"project": []` from a misbehaving proxy) and lets the existing empty-string check on the next line propagate to the `if not project_key or not repo_slug: raise ValueError("missing project key or slug")` path. Behaviour is now: missing-key, null-project, and non-dict-project all surface as the same `ValueError`, which matches the existing error contract the test at `:60-66` (id=`project_non_dict_raises_value_error`) pins down.
- `service/base.py:235-238` — the inline comment is genuinely useful ("Bitbucket Data Center can return `project: null` on orphaned / admin-listed repositories…"). It names the actual API behaviour and points readers at the parallel cloud parser. Good practice — the bug is the kind that gets re-introduced by a "simplifying" refactor 6 months later, and the comment is the cheapest defence.
- `service/resolver.py:95-100` — same pattern, applied to `author`. The `(author_data.get('slug') if isinstance(author_data, dict) else None) or (author_data.get('name') if …) or 'unknown'` chain is verbose but correct: it preserves the original ordering (slug preferred, fall back to name, fall back to literal `'unknown'`) and degrades safely on any non-dict shape. The comment explains *why* the original `.get('author', {})` looked sufficient but wasn't — that explanation is the load-bearing part for future readers.
- `service/resolver.py:96` — the comment correctly identifies that `or 'unknown'` was the *intended* fallback that just never fired because `.get('slug')` raised before reaching the `or`. Spot-on diagnosis.
- `tests/.../test_bitbucket_dc_repos.py:343-385` — `test_parse_repository_handles_null_project` parametrizes three cases:
  - `('project': None, 'slug': 'myrepo')` → expects `ValueError`
  - `('project': [], …)` → expects `ValueError` (non-dict)
  - `({'key': 'PROJ'}, …)` → happy-path → `full_name == 'PROJ/myrepo'`
  Pins down the exact contract the fix establishes. Missing case worth adding: `('project': {'key': None}, …)` — DC has been observed to send empty/null `key` values; current code falls through `project_key = ''` → `ValueError`, which is the right behaviour but isn't pinned by a test.
- `tests/.../test_bitbucket_dc_resolver.py:159-181` — `test_process_raw_comments_handles_null_or_malformed_author` parametrizes five cases including `None`, `{}`, `'not-a-dict'`, name-only, slug-preferred. Excellent coverage — exactly the matrix the fix needs to defend against future regressions. The `'not-a-dict'` case in particular pins the `isinstance` guard that future "simplification" might remove.
- The PR body explicitly defers Forgejo and the cloud Bitbucket `prs.py` `links.html.href` extraction to follow-up PRs "to keep the review surface tight". That's the right call — those have the same shape but different fixtures, and bundling them would make this PR much harder to review for correctness in each integration's specific API quirks.

## Verdict

**merge-as-is**

## Rationale

Two real null-deref bugs, clean idiomatic fixes, comments that explain the *why* in addition to the *what*, and tests that pin the exact contract under five (resolver) and three (parser) shapes. The PR body links the parallel cloud-Bitbucket fixes (#14070, #14085) and is honest about scope — Forgejo and `prs.py` are mentioned as follow-ups rather than silently left out. The optional `('project': {'key': None}, …)` test case is genuinely optional. Nothing here needs another round.
