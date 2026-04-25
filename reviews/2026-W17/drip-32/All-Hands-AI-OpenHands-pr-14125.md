# All-Hands-AI/OpenHands#14125 — fix(integrations): guard Bitbucket Data Center against null nested fields

- PR: https://github.com/All-Hands-AI/OpenHands/pull/14125
- Head SHA: `090a1830979a871984142e43e20ea889ab8fd134`
- Author: boskodev790
- State: OPEN
- Diff: +74 / −3 across
  `openhands/integrations/bitbucket_data_center/service/{base.py,resolver.py}`
  and matching unit tests under
  `tests/unit/integrations/bitbucket_data_center/`.

## Summary

Two crash-on-null sites in the Bitbucket Data Center integration that
mirror cloud-Bitbucket bugs already fixed in #14070 and #14085. Both are
classic Python defensive-default footguns: `dict.get(key, {})` only
short-circuits when the key is *absent*, not when the key is present with
a `None` value, so the chained `.get('subkey')` raises
`AttributeError: 'NoneType' object has no attribute 'get'`. The fix in
both spots is the `(x or {})` idiom plus a defensive
`isinstance(x, dict)` check so non-dict garbage from the upstream API
(arrays, scalars) also degrades gracefully.

## Diff highlights

- `service/base.py:231-243` (`_parse_repository`) — old:
  `project_key = repo.get('project', {}).get('key', '')` raised
  `AttributeError` for `"project": null`. New: extract
  `project = repo.get('project') or {}` then
  `project_key = project.get('key', '') if isinstance(project, dict) else ''`,
  letting the existing `if not project_key or not repo_slug: raise
  ValueError("missing project key or slug")` path fire as designed. The
  semantic improvement is real — the failure is now a typed `ValueError`
  with a clear message, not an unhandled `AttributeError`.
- `service/resolver.py:92-104` (`_process_raw_comments`) — same shape for
  the comment-author fallback. Extracts `author_data = comment_data.get('author')
  or {}` then guards each subsequent `.get('slug')` / `.get('name')` with
  an `isinstance(author_data, dict)` check, so the existing
  `or 'unknown'` fallback fires for null-author comments from
  deleted/anonymous accounts. Without this, *one* null-author comment
  crashes the entire `_process_raw_comments` loop and drops every
  comment after it.
- `tests/.../test_bitbucket_dc_repos.py:341-374` — new parametrized test
  `test_parse_repository_handles_null_project` with three cases:
  `project=None → ValueError`, `project=[] → ValueError`,
  `project={'key':'PROJ'} → 'PROJ/myrepo'`. Asserts the
  `'missing project key or slug'` message explicitly.
- `tests/.../test_bitbucket_dc_resolver.py:159-180` — five-case
  parametrized test covering null author, empty-dict author,
  non-dict author (`'not-a-dict'`), name-only fallback, and slug-preferred
  cases.

## Concerns / Nits

1. **Test fixture realism** is the right call here — using real-shape
   Bitbucket Data Center API payloads (the `parametrize` ids include
   `project_null_raises_value_error`, `author_null` etc.) makes these
   tests load-bearing against future regressions. Worth one more parametrize
   case in the resolver test: `comment_data` itself missing the `'author'`
   key entirely (the original `.get('author', {})` happy path), to pin the
   absent-key behaviour explicitly.
2. **Repeated `isinstance(author_data, dict)` checks** in the resolver
   diff — once you've done `author_data = comment_data.get('author') or {}`,
   the subsequent `isinstance` guards only fire for the third case
   (non-dict scalar), which is rare. The cleaner shape is to coerce up
   front: `author_data = comment_data.get('author') or {}; if not
   isinstance(author_data, dict): author_data = {}`. Functionally
   identical, half the noise downstream.
3. **Sibling fix coverage**: the PR description references #14070 and
   #14085 as cloud-Bitbucket parallels. Worth a follow-up grep across
   the DC integration for any other `.get('x', {}).get('y')` chains —
   places like `get_user`, branch listing, PR listing tend to share this
   bug class. A single grep-and-fix-all PR would be cheaper than landing
   these one-by-one.
4. **No silent data loss**: I confirmed the resolver fix only changes
   the *crash-or-not* outcome, not the contents of returned comments —
   the `or 'unknown'` author label was already the intended behaviour and
   is now reachable.

## Verdict

**merge-as-is** — this is exactly the kind of small, well-tested
defensive fix that should land quickly. The cloud-side parallels are
already merged, the failure mode is a real production crash on real
Bitbucket DC payloads, and the test coverage is the right shape. The
isinstance-coercion nit is purely stylistic.
