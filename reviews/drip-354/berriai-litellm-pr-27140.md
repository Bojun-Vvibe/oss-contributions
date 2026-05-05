# BerriAI/litellm PR #27140 — fix(proxy): union x-litellm-tags with static team/key tags

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/27140
- Head SHA: `b2d1802541630368653b918142d16d0874ca17d9`
- Size: +121 / -1 (one src line, ~120 lines of regression tests)

## Summary

Single behavioral fix in
`litellm/proxy/litellm_pre_call_utils.py:1543-1551`: when
`tags is not None and _admin_allow_client_tags`, the previous code
did

```python
data[_metadata_variable_name]["tags"] = tags
```

…which *overwrote* whatever static admin-controlled tags
(team-level, key-level, project-level) had already been merged into
`data[_metadata_variable_name]["tags"]` earlier in the same
function. The fix calls
`LiteLLMProxyRequestSetup._merge_tags(request_tags=…, tags_to_add=…)`
to union and dedupe.

Plus two genuine integration-style tests in
`tests/test_litellm/proxy/test_litellm_pre_call_utils.py`:

- `test_add_litellm_data_to_request_unions_caller_tags_with_static_key_tags`
  (lines 1028-1086): static key tags `["static-key-tag",
  "shared-tag"]` + caller tags `["caller-tag-1", "caller-tag-2"]`
  + caller-also-supplied `"shared-tag"` should yield all four with
  `shared-tag` deduped to count 1 (line 1088).
- `test_add_litellm_data_to_request_unions_caller_tags_with_static_team_tags`
  (lines 1093-1140): same property at team-metadata level.

## Why this matters

This is a real spend-attribution bug. Imagine a key with admin tag
`["env:prod", "owner:platform-team"]` and an SDK caller that adds
`["feature:new-onboarding"]` for per-feature spend tracking. Before
this PR, enabling `allow_client_tags=true` to pick up the feature
tag would silently *erase* the env/owner tags from the spend log,
and the operator gets a daily report that says "your platform-team
budget is empty" because every caller-tagged request now lacks the
team tag entirely. That's the kind of bug nobody notices until the
finance review.

## What's good

- The fix is exactly one line in the production path. No
  refactoring, no scope creep. The PR's risk surface is the size
  of `_merge_tags`'s contract.
- Both test cases assert *positive presence* of all four expected
  tags AND the deduplication property
  (`assert final_tags.count("shared-tag") == 1`, line 1088). That's
  the right shape — an `assertCountEqual` would have hidden the
  dedup property.
- The team-metadata test (lines 1093-1140) is not redundant: the
  static-tag merge happens in different code paths for `metadata`
  vs `team_metadata`, and the bug could plausibly have existed at
  one level only.
- The inline comment in `litellm_pre_call_utils.py:1544-1549` is
  *explanatory*, not just descriptive — it spells out why plain
  assignment was wrong ("forcing an all-or-nothing trade-off") and
  why `_merge_tags` is safe ("deduplicates, so callers cannot
  poison the list").

## Concerns / nits

1. **`_merge_tags` contract assumed but not verified.** The fix
   assumes `_merge_tags` returns a `list[str]`, dedupes, and
   handles `None` for `request_tags`. If `_merge_tags` happens to
   sort or reorder, downstream consumers that do positional things
   with `tags[0]` will see a behavioral shift. Worth a one-line
   reference in the comment to where `_merge_tags` is defined and
   what its dedup ordering guarantee is (FIFO? sorted?).
2. **No test for the `elif tags is not None` branch** (line 1553+
   in the original file). That's the opposite branch — caller
   supplied tags but `_admin_allow_client_tags=False` — and it
   logs a warning and ignores. A negative test confirming the
   static tags are *preserved unchanged* would round out the
   coverage.
3. **Spend-log integration test is missing.** This is a property
   about what ends up in `LiteLLM_SpendLogs.request_tags`, not
   just `data["metadata"]["tags"]`. A higher-level test that
   actually issues a `/chat/completions` and reads back the spend
   log would be the gold standard, but that's a bigger refactor
   and not blocking.
4. **Tag-poisoning angle.** The comment claims "callers cannot
   poison the list" because `_merge_tags` dedupes. That's true
   for *exact* duplicates, but a malicious caller could still
   submit `["env:prod\n", "env:prod "]` (whitespace variants) and
   inflate cardinality on whatever telemetry sink consumes
   `request_tags`. Out of scope for this PR but worth tracking
   somewhere — sanitization belongs upstream of merge.

## Verdict

**merge-after-nits** — the fix is correct, minimal, well-commented,
and the tests pin down the deduplication property. Add a brief
reference to `_merge_tags`'s contract in the comment and ideally a
negative test for the `allow_client_tags=False` branch. This is
exactly the kind of one-line behavioral fix that should ship with
a backport note: anyone running with `allow_client_tags=true` on a
prior release has been silently losing static admin tags.
