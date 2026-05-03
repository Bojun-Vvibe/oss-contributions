# Review: BerriAI/litellm #26982 — fix(gemini): avoid duplicate model route for full api_base

- PR: https://github.com/BerriAI/litellm/pull/26982
- Head SHA: `f981e4abb0f4890e5cef4963b711e7511bf458af`
- Author: StatPan
- Size: +39400 / -1615 (large because the PR retargets a fix through
  the OSS staging path; see body — the *intended* fix is small)

## Summary

The advertised change is a 10-line fix in
`litellm/llms/vertex_ai/vertex_llm_base.py` that stops appending a
duplicate `/models/{model}:{endpoint}` segment when a custom Gemini
`api_base` already includes that route. Three new parametrised tests
cover (a) `api_base` already terminating in
`/models/{model}:{endpoint}`, (b) terminating in `/models/{model}`
(needs only `:endpoint` appended), and (c) a clean prefix that needs
the full `/models/{model}:{endpoint}` suffix. The PR body notes this
is a re-route of an earlier direct-to-`main` PR (#26980) through
`litellm_internal_staging` so the main-branch source guard accepts
it.

The +39k/-1.6k line count comes from staging-branch sync content —
unrelated work for redis TTL refresh, MCP DB credentials,
async_increment_cache plumbing, voyage rerank changes, an
openapi-snapshot CI workflow, and many tests. None of those are
called out in the PR title or summary.

## Specific citations

- `litellm/llms/vertex_ai/vertex_llm_base.py:412-422` — the actual
  fix. Three-branch dispatch:
  - if `api_base.endswith(f"/models/{model}:{endpoint}")` → use
    `api_base` as-is.
  - elif `api_base.endswith(f"/models/{model}")` → append
    `f":{endpoint}"`.
  - else → `"{}/models/{}:{}".format(api_base.rstrip("/"), model,
    endpoint)`. Note the `rstrip("/")` — fixes the secondary bug
    where a trailing slash on `api_base` would have produced
    `.../v1beta//models/...`.
- `tests/.../test_vertex_llm_base.py:891-928` (the
  `test_check_custom_proxy_gemini_prebuilt_route_handling`
  parametrize block) — three cases that exactly mirror the three
  branches. Good direct coverage of the new logic.
- `litellm/caching/dual_cache.py:392-419` and
  `litellm/caching/redis_cache.py:824-858` — unrelated to the gemini
  fix. Adds a `refresh_ttl: bool = False` parameter that flips the
  Redis TTL semantics from "set TTL only on first write" to "refresh
  on every write". Default `False` preserves existing window-style
  behaviour; this is a backward-compatible additive change but it's
  policy-sensitive (think: rate limiters, session caches) and
  belongs in its own focused PR with its own justification.
- `.github/workflows/check-lazy-openapi-snapshot.yml` (+75) — new CI
  workflow that diffs `litellm/proxy/_lazy_openapi_snapshot.json`
  against a freshly regenerated copy and posts a `neutral` check
  conclusion when the snapshot is stale. Sensible CI hygiene, again
  unrelated to the gemini fix.
- `litellm/litellm_core_utils/initialize_dynamic_callback_params.py:23-34`
  — extracts `validate_no_callback_env_reference` helper. Pure
  refactor. Unrelated.
- `litellm/constants.py:1428` — adds
  `CLI_SSO_SESSION_TTL_SECONDS = 600`. Unrelated.

## Verdict

**request-changes**

## Rationale

The advertised fix is correct, narrow, and well-tested. Endpoint URL
construction for proxied Gemini is exactly the sort of place
defensive `endswith` matching makes sense, the `rstrip("/")` catches
a real adjacent bug, and the three parametrized tests fully cover
the branches. If the diff were just those ~12 lines plus the test, I
would have voted `merge-as-is`.

The reason for `request-changes` is the **diff hygiene**. A PR
titled "fix(gemini): avoid duplicate model route for full api_base"
should not contain:

- a Redis TTL semantics change (`refresh_ttl` parameter cascading
  through `dual_cache` and `redis_cache`),
- a new CI workflow for an openapi snapshot,
- a 400-line MCP DB-credentials test file,
- a callback-env validation refactor,
- a CLI SSO TTL constant,
- and many more unrelated edits totalling tens of thousands of lines.

The PR body explains the situation (rerouted via staging because of
a main-branch source guard), but rerouting through staging shouldn't
mean folding in every other staging change. This PR should be
rebased so it contains only the gemini URL fix and its tests; the
other changes belong in their own focused PRs (some of which may
already exist on staging and just need to be promoted independently).

Beyond hygiene: each of the unrelated changes carries its own review
risk that gets lost in a 39k-line diff. The Redis `refresh_ttl`
change in particular modifies a well-trodden path (rate-limiting,
budget tracking) and deserves a focused reviewer who specifically
thinks about TTL semantics, not a reviewer skimming for the gemini
URL bug.

If the maintainers can't easily split this (e.g. because staging is
the merge target and these are accumulated commits), at minimum the
PR title should accurately reflect the scope and the description
should enumerate every behavioural change so reviewers know what to
look at. As-is, an auto-merge would land six unrelated changes
under a misleading commit message.

## Required changes

1. Rebase to contain only the
   `litellm/llms/vertex_ai/vertex_llm_base.py` fix and the new
   parametrized test block. Open separate PRs for everything else.
2. If splitting truly isn't possible, retitle to something like
   `chore(staging): forward staging changes including gemini api_base
   fix` and enumerate every behavioural change in the description.
3. Add a code comment near `vertex_llm_base.py:412` explaining why
   `endswith` (substring matching) is safe here — namely, the model
   name is part of the suffix so there's no risk of matching a
   prefix that happens to share the tail.
