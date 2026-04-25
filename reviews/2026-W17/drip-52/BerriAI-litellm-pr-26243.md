---
pr: 26243
repo: BerriAI/litellm
sha: 3bd76e2aa04f9ec72a6a67056fae5fb18417c61b
verdict: merge-after-nits
date: 2026-04-26
---

# BerriAI/litellm#26243 — fix: respect drop_params when mapping metadata.user_id to user in Responses adapter

- **URL**: https://github.com/BerriAI/litellm/pull/26243
- **Author**: Kcstring
- **Files**:
  `litellm/llms/anthropic/experimental_pass_through/responses_adapters/transformation.py`,
  `tests/test_litellm/llms/anthropic/experimental_pass_through/responses_adapters/test_responses_adapters_transformation.py`
  (+52/-2)

## Summary

The Anthropic→Responses adapter unconditionally mapped
`metadata.user_id` (Anthropic input) into the OpenAI Responses
`user` field. For deployments that set `litellm.drop_params=True`
because their backend (e.g. an OpenAI-compatible proxy that doesn't
recognize `user`) rejects unknown params, this caused the request
to fail. The PR gates the mapping on `litellm.drop_params` so
`user` is omitted when drop_params is set. Closes #26241.

## Reviewable points

- `transformation.py:386-389` — the entire change is two lines:

  ```python
  if isinstance(metadata, dict) and "user_id" in metadata:
      if not litellm.drop_params:
          responses_kwargs["user"] = str(metadata["user_id"])[:64]
  ```

  Semantically correct. The `[:64]` clamp on the right-hand side
  is preserved (OpenAI's documented max for `user`). The
  drop_params gate is the lone new condition.

- The test class `TestDropParamsMetadataUserId`
  (`test_responses_adapters_transformation.py:1046-1092`) covers
  three cases: drop_params=False (user is set), drop_params=True
  (user is omitted), and no metadata (user is never set). The
  try/finally pattern around `litellm.drop_params = ...` properly
  restores global state — important because pytest runs tests in
  the same process and a leaked True would poison every later
  test in the file.

## Risks / nits

- **Nit**: the new `import litellm` at the top of
  `transformation.py:11` (line numbers in the diff at line 9-10)
  is added to a module that already imports many `litellm.*`
  submodules. Scanning the existing file would clarify whether
  the pattern there is to import the package or the submodules
  individually — fine either way, but worth a one-line check
  during review.

- **Nit**: `drop_params` is a per-request-overridable setting in
  the rest of the codebase (callers can pass `drop_params=True`
  to a single completion call to override the global). This PR
  only reads `litellm.drop_params` (the module-global) and
  doesn't accept a per-request override. That matches the
  surrounding adapter code, which doesn't thread per-request
  config through `translate_request`, so it's consistent — but if
  the adapter ever grows that surface, this gate should follow.

- **Nit**: the omission is silent. A user who sets
  `drop_params=True` and *did* want `user` to flow through (just
  not via metadata) gets no warning. Acceptable: that's the whole
  point of `drop_params` everywhere else in litellm — silent
  drop. Consistent.

- The `metadata.user_id` → `user` mapping is itself a slightly
  surprising contract; it's documented in the adapter but anyone
  reading downstream traces will still see `user="alice"` and
  wonder where it came from. Out of scope for this PR.

## Verdict

`merge-after-nits`. The fix is correct, minimal, and
well-tested. Verify the import style nit aligns with file
conventions; otherwise ship.

## What I learned

When an adapter implicitly *adds* a parameter (rather than passing
through what the caller sent), the `drop_params` policy should
gate the *addition* too — otherwise you've quietly broken the
contract that "drop_params=True means the request only contains
what backend accepts". This PR closes that gap for one mapping;
the same pattern likely lives in other Anthropic→Responses field
translations and is worth a sweep.
