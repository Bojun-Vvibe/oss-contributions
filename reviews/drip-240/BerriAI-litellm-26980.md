# BerriAI/litellm#26980 — fix(gemini): avoid duplicate model route (duplicate of #26982)

- **PR**: https://github.com/BerriAI/litellm/pull/26980
- **Head SHA**: `f981e4abb0f4890e5cef4963b711e7511bf458af`
- **Verdict**: `needs-discussion`

## What it does

This PR carries the *identical* head SHA
(`f981e4abb0f4890e5cef4963b711e7511bf458af`) as #26982, and the diff is
the same set of files: `litellm/llms/vertex_ai/vertex_llm_base.py`
(+8/-1), `litellm/router_strategy/tag_based_routing.py` (+2/-1), plus
the two test files (+41/-5 and +28/-0). Same author (StatPan), same
PR-body content, same `updatedAt` timestamp within seconds.

## What's load-bearing

- The diff is structurally identical to #26982 — see
  `BerriAI-litellm-26982.md` in this drip for the line-level review of
  `vertex_llm_base.py:412-422` (three-branch URL construction:
  exact-match passthrough → model-prefix `:endpoint` append → fallback
  with `.rstrip("/")`) and the bundled tag-routing tightening at
  `tag_based_routing.py:109` (`deployment_has_plain_tags` named
  predicate replacing `bool(deployment_tags)`).
- Both PRs target the same base branch and have the same head ref OID,
  meaning they're either two PRs pointing at the same fork branch (one
  is a duplicate that should be closed), or there's a CI/automation
  glitch that double-opened the same change.

## Concerns

1. **Duplicate of #26982 — one of the two should be closed.** Carrying
   two open PRs with identical diffs against the same base branch
   wastes maintainer review cycles, doubles the CI surface, and creates
   merge ambiguity (whichever lands first invalidates the other). The
   maintainers should close whichever is the duplicate (typically the
   higher-numbered, but since these have an identical SHA it's clearly
   an automation artifact rather than a deliberate second submission).
2. **All technical concerns from #26982 apply equally** — see that
   review for the (a) bundled-tag-routing-not-mentioned-in-PR-body
   concern, (b) `.rstrip("/")` as a fix-within-the-fix worth calling
   out, (c) `?query` suffix edge case in the `.endswith` check, and (d)
   restored `assert` in `test_credential_project_validation` deserving
   a PR-body note.
3. **No way to tell from the GitHub API which is the canonical PR.**
   `headRefOid`, `updatedAt`, and `additions/deletions` all match. The
   PR body on both is identical down to the OpenHands cross-references.
   Maintainers will have to ask the author to close the duplicate.

## Verdict

`needs-discussion` — the change itself is correct (identical to #26982,
which is `merge-after-nits`), but having two PRs open for the exact
same change is a process problem that needs author/maintainer
coordination before either can land cleanly. Once one is closed, the
remaining one becomes `merge-after-nits` per the #26982 review.
