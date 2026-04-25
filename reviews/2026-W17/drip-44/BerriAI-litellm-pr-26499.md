# BerriAI/litellm PR #26499 — fix(auth): join team-member budget so rpm/tpm limits are enforced

- **URL:** https://github.com/BerriAI/litellm/pull/26499
- **Author:** @kiyeonjeon21
- **State:** OPEN (base `litellm_internal_staging`)
- **Head SHA:** `9f1e970`
- **Size:** +851 / -3 (most additions are in `model_prices_and_context_window_backup.json`)

## Summary of change

Title is `fix(auth): join team-member budget so rpm/tpm limits are
enforced`. The PR's actual change scope spans two unrelated areas:

1. **Auth / budget enforcement**: a small set of edits in the
   user-API-key auth path that join the team-member budget row when
   resolving rpm/tpm limits, so a team-member with a per-member budget
   has those limits enforced (rather than only the parent team budget).
2. **Model price catalog**: additions of `azure/gpt-5.5`,
   `azure/gpt-5.5-2026-04-23`, `azure/gpt-5.5-pro`,
   `azure/gpt-5.5-pro-2026-04-23`, plus `gpt-5.5` /
   `gpt-5.5-2026-04-23` updates to
   `model_prices_and_context_window_backup.json`. ~163 lines of pricing
   metadata: input/output cost tiers (above_272k_tokens, flex,
   priority), `max_input_tokens` raised to 1,050,000,
   `supports_xhigh_reasoning_effort: true`, etc.

## Findings against the diff

- **Scope conflation.** A "fix(auth)" titled PR shipping ~700 lines of
  pricing-table additions is going to be hard to review well. The
  pricing table is a separate concern — typically updated via a
  pricing-only PR or by a model-catalog automation job — and merging
  it through an auth-fix branch makes both git blame and revert harder.
  Strongly recommend splitting before merge.
- **Pricing entries internal consistency**: `azure/gpt-5.5` declares
  `supports_minimal_reasoning_effort: false` but the bare `gpt-5.5`
  entry declares `supports_minimal_reasoning_effort: true`. The two
  rows are otherwise twins (same input/output costs, same context
  window). This may be intentional (Azure's surface differs from
  OpenAI's), but it deserves a one-line comment or a test pin so a
  later sweep doesn't "fix" the inconsistency in the wrong direction.
- **`max_input_tokens` jump from 272,000 → 1,050,000** for `gpt-5.5`:
  this is the kind of capacity-tier change that has cascading effects
  on retry / chunking / fallback decisions elsewhere. Worth confirming
  the relevant `BaseLLM.get_max_input_tokens()` consumer paths read
  from this catalog rather than hard-coding 272k.
- **Base branch is `litellm_internal_staging`**, not `main`. That's
  consistent with the BerriAI workflow of staging large catalog
  updates, but it means any reviewer should land the auth fix portion
  *separately* against `main` so the rpm/tpm regression doesn't sit
  behind the catalog update.
- **The auth diff itself is not visible in the head-200 of the diff
  output** (the catalog churn dominates). Need to view the full
  diff to assess the join SQL change quality.

## Verdict

**needs-discussion**

The auth fix is plausibly correct and almost certainly needed, but
the PR as currently structured is not reviewable in one pass:

1. Split into two PRs:
   - `fix(auth): join team-member budget…` — the auth-only diff.
   - `chore(model_prices): add gpt-5.5 / gpt-5.5-pro` — the catalog
     update.
2. For the catalog PR, reconcile the
   `supports_minimal_reasoning_effort` discrepancy between
   `azure/gpt-5.5` (false) and `gpt-5.5` (true), or add a comment
   explaining why the surface diverges.
3. For the auth PR, ensure the JOIN doesn't change the row count of
   the parent query in the no-team-member case (regression risk for
   single-user keys).

The base-branch choice (`litellm_internal_staging`) suggests the
maintainers are aware this is a staging PR; they should still split
it before promoting to `main`.

## What I learned

"One PR, one concern" is cheap to say and frequently violated in
LLM-infrastructure repos because pricing-catalog churn is so constant
that contributors tack it onto whatever branch they happen to have
open. The cost is paid at review and at revert time: a regression
in either half forces the maintainer to choose between reverting
both or doing surgical revert-with-cherry-pick. The healthier policy
is a separate `model_prices` automation that opens its own PRs on a
schedule, removing the temptation to bundle.
