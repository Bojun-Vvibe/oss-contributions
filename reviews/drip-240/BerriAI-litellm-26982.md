# BerriAI/litellm#26982 — fix(gemini): avoid duplicate model route for full api_base

- **PR**: https://github.com/BerriAI/litellm/pull/26982
- **Head SHA**: `f981e4abb0f4890e5cef4963b711e7511bf458af`
- **Verdict**: `merge-after-nits`

## What it does

Fixes a Gemini-routing bug in `vertex_llm_base.py:_check_custom_proxy`
where any caller setting `api_base` to a URL that already ended in
`/models/{model}:{endpoint}` (or even just `/models/{model}`) would get
the route appended *again*, producing nonsense like
`.../models/gemini-2.5-flash-lite:generateContent/models/gemini-2.5-flash-lite:generateContent`.

The fix adds a three-arm branch detecting full-route, model-prefix, and
fallback cases. +49 / -7. Closes #26979 and unblocks downstream
OpenHands integrations OpenHands/OpenHands#14028 / #14029. Replaces an
inactive prior PR (#24786, dormant since 2026-03-30).

## What's load-bearing

- `litellm/llms/vertex_ai/vertex_llm_base.py:412-422`:
  ```python
  if api_base.endswith(f"/models/{model}:{endpoint}"):
      url = api_base
  elif api_base.endswith(f"/models/{model}"):
      url = f"{api_base}:{endpoint}"
  else:
      url = "{}/models/{}:{}".format(api_base.rstrip("/"), model, endpoint)
  ```
  Three branches in the right order: full-route exact-match first
  (idempotent passthrough), then model-prefix (append only `:{endpoint}`),
  then the prior fallback construction with `.rstrip("/")` correctly
  added so a trailing-slash `api_base` no longer produces a `//models/`
  double-slash.
- `tests/test_litellm/llms/vertex_ai/test_vertex_llm_base.py:891-928` —
  three parametrized cases pinning each branch:
  full-route → idempotent, model-prefix → `:endpoint` appended,
  trailing-slash → no double-slash. These are the right three arms; one
  per branch.
- `tests/test_litellm/router_strategy/test_router_tag_routing.py:349-374`
  adds `test_strict_tag_routing_without_request_tags_blocks_header_regex_fallback`
  pinning the sister tag-routing fix at
  `litellm/router_strategy/tag_based_routing.py:109` where
  `strict_tag_check_failed = not match_any and bool(deployment_tags)` was
  flipped to `not match_any and deployment_has_plain_tags` (an explicit
  `is not None and len(...) > 0` check). This catches the case where a
  deployment with `tag_regex` but no plain `tags` could be matched by a
  spoofed `User-Agent` header on a request with no request tags.

## Concerns / nits

1. **Two unrelated changes in one PR.** The Gemini api_base fix and the
   tag-routing strict-check tightening are bundled. The PR title and
   summary speak only to the Gemini fix; the tag-routing change is
   described only by an embedded merge commit
   (`Merge pull request #26805 from .../litellm_auth_bypass_tag_based_routing`).
   This is cherry-picked from a sister PR and is a security tightening,
   so reviewers should know it's there. Either split, or add a
   "Bundled changes" section to the PR body.
2. **The fallback branch's `.rstrip("/")` is itself a behavior change for
   the existing path.** Pre-fix: `api_base="https://x.com/v1/"` produced
   `https://x.com/v1//models/...:generateContent` (broken double-slash);
   post-fix: produces clean `.../v1/models/...:generateContent`. This is
   a fix-within-the-fix. Worth one line in the PR body and one
   parametrized test arm pinning it (the third existing parametrized
   case at `:907-911` exercises trailing-slash → no double-slash, which
   covers it but isn't called out in the test name).
3. **`api_base.endswith(f"/models/{model}:{endpoint}")` is case-sensitive
   and exact.** If a user passes `api_base` with a trailing `?param=...`
   query string (common in proxy URLs that authenticate via query), the
   `.endswith` check misses and the fallback produces nonsense. Probably
   out of scope for this PR but worth a TODO comment at `:412`.
4. **No `test_credential_project_validation` justification for the
   restored assertion at `:71` (`assert result == ("fake-token-1",
   "different-project")`)** — the test previously had `print(result)`
   and now asserts a specific tuple. Per the commit message
   ("test(vertex): restore credential project validation expectation"),
   this is restoring a regression the prior PR removed. Worth a one-line
   PR-body note that this PR also fixes a test-quality regression.

## Verdict

`merge-after-nits` — the core fix is correct and well-tested across all
three branches, and the bundled tag-routing security tightening is
genuinely valuable (closes a header-regex auth-bypass primitive). Please
split or call out the bundled changes in the PR body before merge.
