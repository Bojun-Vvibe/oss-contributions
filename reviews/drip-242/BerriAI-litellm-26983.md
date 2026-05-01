# Review: BerriAI/litellm #26983 — fix(gemini): avoid duplicate model route for full api_base

- URL: https://github.com/BerriAI/litellm/pull/26983
- Head SHA: `562a1452a19edbf4eb9312681b363255c75fd3aa`
- Files: 2 (`litellm/llms/vertex_ai/vertex_llm_base.py`,
  `tests/test_litellm/llms/vertex_ai/test_vertex_llm_base.py`)
- Size: +49/-6

## Summary of intended change

Replaces the unconditional
`url = "{}/models/{}:{}".format(api_base, model, endpoint)` at
`vertex_llm_base.py:415` (pre-PR) with a three-branch construction at
`:415-422` that detects how much of the route the caller's `api_base`
already contains:

1. **Full route already present** (`api_base` ends with
   `/models/{model}:{endpoint}`) → use `api_base` as-is. Idempotent
   passthrough — fixes the duplicate-route shape
   `.../models/X:generateContent/models/X:generateContent`.
2. **Model prefix present** (`api_base` ends with `/models/{model}`)
   → append only `:{endpoint}`.
3. **Fallback** → strip a trailing `/` from `api_base` then append the
   full `/models/{model}:{endpoint}`. The new `.rstrip("/")` is the
   fix-within-the-fix that closes the
   `https://proxy/v1beta//models/...` double-slash shape.

Plus three parametrized regression cases at
`test_vertex_llm_base.py:891-928` covering each branch, and a small
test-file cleanup (drops `unittest.mock.call`, drops `litellm` /
`DEFAULT_MAX_RECURSE_DEPTH` unused imports, replaces a `print(...)` at
`test_credential_project_validation:71` with the proper assertion
`assert result == ("fake-token-1", "different-project")`).

## Review

### Duplicate-of-#26982 / #26980

This PR is the **third copy** of the same fix. From the head SHAs:
- `#26980` head `f981e4abb0f4890e5cef4963b711e7511bf458af`
- `#26982` head `f981e4abb0f4890e5cef4963b711e7511bf458af`
- `#26983` head `562a1452a19edbf4eb9312681b363255c75fd3aa` ← *this*

`#26980` and `#26982` shared an identical SHA (clearly an automation
double-submit). `#26983` has a *different* SHA but identical file list,
identical PR body title, identical change shape. Either it's a force-push
of the same author's branch under a fresh PR number (the most likely
shape — author may have re-tried after the prior duplicates went stale),
or it's a third independent copy with subtle diffs.

The diff itself is **functionally identical** to the
`vertex_llm_base.py:412-422` change reviewed in drip-240 against #26982:
same three-branch control flow, same `.rstrip("/")`, same test cases
verbatim. The test-file cleanup (`call` import removal, `litellm` import
removal, `assert` replacing `print`) is **also** present in this PR but I
do not have the same level of confirmation it was in #26982 — this looks
like an additive cleanup either way, not a divergence.

### Technical review (for the change itself)

Already covered in detail in drip-240/BerriAI-litellm-26982.md; the same
verdict applies to the code:

- Branch ordering is correct (most-specific first, fallback last) so a
  caller passing exactly `.../models/{model}:{endpoint}` doesn't fall
  through to the `:{endpoint}` append branch.
- `.rstrip("/")` only on the fallback branch means a caller who passes
  `.../models/{model}/` (trailing slash on the model-prefix shape) will
  *not* be normalized — they hit branch 2 with literal trailing slash,
  producing `.../models/{model}/:{endpoint}`. The
  `endswith(f"/models/{model}")` check is exact-match so that case
  *won't* match branch 2 either; it falls through to fallback where
  `.rstrip("/")` does the right thing. So the trailing-slash variant is
  handled, just by routing through a different branch. Worth a comment.
- `?query=` suffix on `api_base` (e.g. `?key=...`): the `endswith` checks
  in branches 1 and 2 will both fail, falling through to fallback which
  appends `/models/X:generateContent` *after* the query string,
  producing `...?key=...?/models/X:generateContent` — broken URL.
  Pre-PR shape was the same, so this isn't a regression, but worth a
  TODO comment at `:412` noting that callers who pass query-string
  api_bases will still break.

### Tests

The three parametrized cases at `:891-928` cover each branch, and the
expected URLs are stated literally so future regressions are obvious.
The `test_credential_project_validation` cleanup at `:71` (replacing
`print(f"result: {result}")` with
`assert result == ("fake-token-1", "different-project")`) is a tiny but
real strengthening of an unrelated test that previously was just
formatting output and not asserting — clean drive-by improvement.

What's not covered: the `:` separator inside the model name itself
(`gemini-2.5-flash-lite-preview-04-17:streamGenerateContent` would
match `endswith(f"/models/{model}:{endpoint}")` only if endpoint is
`streamGenerateContent` — but if a caller passes `model =
"gemini-2.5-flash-lite-preview-04-17"` and `endpoint =
"streamGenerateContent"` against an `api_base` constructed for
`generateContent`, branch 1 fails and branch 2 also fails, falling
through to fallback — a copy-of-route-with-wrong-endpoint scenario.
Probably a non-issue in practice; out of scope.

## Verdict

**needs-discussion**

Not for the technical change — the fix is correct and matches the
already-reviewed #26982. **For the duplication itself**: with three open
PRs all carrying the same fix, only one should land. Recommend the author
or a maintainer:

1. Pick one (this one if the SHA is the latest revision with the
   test cleanup; otherwise close in favor of #26982).
2. Close the other two with a comment pointing at the chosen PR.
3. Squash so the merge commit doesn't carry three sets of "open new PR"
   noise into history.

Once consolidated, the chosen PR is merge-after-nits with the same
follow-ups as #26982 (PR body callout for the `.rstrip("/")` fix-within-
fix, TODO at `:412` for the `?query=` shape, brief comment at branch 2
explaining that trailing-slash model-prefix shapes intentionally fall
through to fallback rather than matching branch 2).
