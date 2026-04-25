# BerriAI/litellm PR #26455 — feat: per-model team member budgets

- **URL:** https://github.com/BerriAI/litellm/pull/26455
- **Author:** @ishaan-berri
- **State:** OPEN (targets `litellm_internal_staging`)
- **Head SHA:** `6c918acf285673b81365842a82071610a517ba76`

## Summary of change

Two-part PR (the title is misleading — most of the diff is rebased
content from #26449's GPT-5.5 day-0 work):

1. **Carries** the `gpt-5.5` / `gpt-5.5-pro` registry additions from
   #26449 (already reviewed there).
2. **New routing change** in `litellm/proxy/_types.py`:
   - Adds `/project/list` and `/project/info` to
     `LiteLLMRoutes.self_managed_routes` (~L681) so endpoints can
     scope their own results by caller team.
   - Adds the same two paths to the org-admin route allowlist (~L714).

## Findings against the diff

- **`_types.py` L681–684:**
  ```python
  # Project routes - endpoints scope results to caller's teams (non-admin gets only their projects)
  "/project/list",
  "/project/info",
  ```
  Placing these in `self_managed_routes` is correct *only if* the
  handler implementations actually filter by caller team. The PR
  diff in this view does not include the handler code — that's a
  **request-changes blocker** unless a separate hunk adds the
  filtering logic. Without it, anyone with a valid key gets all
  projects.
- **L717–718 (org-admin allowlist):** adding `/project/list,
  /project/info` here is fine and additive.
- **`info_routes` reuse:** the second hunk uses `+ info_routes` —
  `/project/info` reading like a per-resource info route is
  consistent with the existing `/key/info`, `/team/info` shape, so
  no naming concern.
- **PR title vs. content mismatch:** the title still says "per-model
  team member budgets" but I see *no* budget logic in the visible
  diff — only the project route registration plus the rebased model
  registry. This is likely a stale title from when the branch was
  first pushed. The maintainer should retitle before merge to avoid
  changelog confusion (related stale PRs #26462, #26465, #26471
  with the same title were already closed).
- **Branch hygiene:** targets `litellm_internal_staging` not `main`,
  so it'll get squashed into a staging merge — typical for this
  repo.

## Verdict

**request-changes**

Two issues that should be addressed before merge:

1. **Surface the handler code** that actually scopes
   `/project/list` / `/project/info` by caller team. If it lives
   in another PR/commit, link it; if it's missing, this opens a
   data-leak window (any key sees all projects).
2. **Retitle** the PR — the visible scope is "register
   /project/list+/project/info as self-managed + carry GPT-5.5
   registry rebase," not "per-model team member budgets."

The model-registry portion itself is fine and was already approved
via #26449.
