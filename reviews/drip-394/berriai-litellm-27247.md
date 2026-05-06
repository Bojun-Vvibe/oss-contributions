# Review: BerriAI/litellm #27247 — [Fix] Union x-litellm-tags with static team/key tags

- Head SHA: `0c07365b6239234eae35f241bcbc8589efda6847`
- Files: `litellm/proxy/litellm_pre_call_utils.py`, `tests/test_litellm/proxy/test_litellm_pre_call_utils.py`

## Summary

Closes a real silent-overwrite bug in the proxy's caller-tag handling. When
a key/team had `allow_client_tags=True` and a caller passed
`x-litellm-tags: tenant:1681` in headers, the proxy was *replacing* the
key/team's static `tags` with just the caller-supplied list, instead of
unioning them. This silently dropped operator-configured tags
(`team:platform`, `env:prod`) from spend logs and routing, breaking
billing-by-tag and tag-based routing for any team that allowed client tags.

## Specifics

- `litellm/proxy/litellm_pre_call_utils.py:1667-1673` — the load-bearing
  one-line bug fix:

  ```python
  # before: data[_metadata_variable_name]["tags"] = tags
  # after:
  data[_metadata_variable_name]["tags"] = LiteLLMProxyRequestSetup._merge_tags(
      request_tags=data[_metadata_variable_name].get("tags"),
      tags_to_add=tags,
  )
  ```

  Routes the caller-supplied `tags` through the existing
  `LiteLLMProxyRequestSetup._merge_tags(...)` helper (which is the same
  helper already used elsewhere in the file for the equivalent
  team/key/header merge in the no-`allow_client_tags` branch). Centralizing
  on one merge function means dedup + ordering semantics are now consistent
  across both branches.

- `tests/test_litellm/proxy/test_litellm_pre_call_utils.py:1146-1199` —
  `test_add_litellm_data_to_request_unions_caller_header_tags_with_static_key_tags`
  pins the key-level case: static tags `["team:platform", "env:prod"]` +
  caller header `x-litellm-tags: tenant:1681` → all three present in
  `updated["metadata"]["tags"]`. Asserts via `in` (so order-insensitive).

- `tests/.../test_litellm_pre_call_utils.py:1201-1247` —
  `..._with_static_team_tags` mirrors the same contract for team-level
  static tags (`team_metadata.tags`).

- `tests/.../test_litellm_pre_call_utils.py:1249-1294` —
  `..._dedups_overlapping_caller_and_static_tags` asserts
  `final_tags.count("env:prod") == 1` for the case where caller header
  and static tag overlap. This is the test that pins `_merge_tags` is doing
  the right thing on dedup, and is the one that would catch the most likely
  regression (a future contributor switching to plain list `+` would pass
  the first two tests and fail this one).

## Nits

- `_merge_tags` is referenced as a `@classmethod` / `@staticmethod` of
  `LiteLLMProxyRequestSetup` here but its definition is not in the visible
  diff. Worth confirming it preserves order-of-first-occurrence (rather
  than sorting) so static dashboards that expect a stable tag order in
  spend logs don't see a churn-on-deploy after this lands.
- The `elif tags is not None:` branch at `:1670+` (caller supplied tags
  but `allow_client_tags=False`) still just `verbose_proxy_logger.warning`s
  and drops them. Worth a metric counter increment so SREs can detect
  callers that are spamming `x-litellm-tags` against teams that don't
  permit it.
- Three near-duplicate test bodies (≈ 50 lines each) that differ only in
  the `metadata` vs `team_metadata` placement and the dedup assertion —
  parametrize-friendly. `pytest.mark.parametrize("static_location, ...",
  [...])` would shrink this to one ~30-line test and prevent drift if a
  fourth case (e.g. *both* key-level and team-level static tags) is
  added later.

## Verdict

`merge-after-nits`
