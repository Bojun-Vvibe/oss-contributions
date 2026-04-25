# sst/opencode PR #24330 — fix broken CI workflows and infra migration

@c383e2b2 · base `main` · +1/-1 (per visible diff; PR body lists 3 changes total) · author `alfredocristofano`

## Summary
Closes #24326. Three CI/infra papercuts bundled into one PR per the description:
1. `nix-eval.yml` pinned to `actions/checkout@v6`, which doesn't exist — should be `@v4`.
2. `docs-update.yml` referenced the old repo slug, fixed to the post-rename slug.
3. `infra/app.ts` migration block had `oldTag` and `newTag` set to identical values, making the Durable Objects migration a silent no-op.

## What changed
- `infra/app.ts:44` — flips:
  ```ts
  newTag: $app.stage === "production" || $app.stage === "thdxr" ? "" : "v1",
  ```
  to a hard-coded:
  ```ts
  newTag: "v2",
  ```
  while leaving `oldTag` keyed on stage. The leading comment already says *"when releasing the next tag, make sure all stages use tag v2"* — this PR is that release.
- `nix-eval.yml` and `docs-update.yml` changes called out in the body but not in the visible diff window — recommend reviewer pulls those locally to confirm.

## Key observations
- The `app.ts` change is the highest-stakes piece: in production this drives the Durable Objects rename migration. Once `newTag: "v2"` lands and is deployed, the `oldTag` mapping becomes meaningful for non-prod stages (they migrate from `v1` → `v2`); production migrates from `""` → `v2`. That's exactly the rollout shape the comment describes.
- Bundling a one-character workflow typo (`@v6` → `@v4`), a repo-rename string update, and a live DO-migration tag flip into a single PR is risky for revert: if production has any issue with the migration, you can't cleanly revert without also reverting the workflow fixes. Strongly suggest splitting the `infra/app.ts` change into its own PR.
- No test / dry-run output is referenced. For a DO migration, even a `sst diff` snippet showing the planned migration in the PR description would build confidence.
- The repo-rename string fix is the kind of change that should sweep the whole tree via grep, not just `docs-update.yml`. Confirm there are no other workflow files or docs still pointing at the old slug.

## Risks/nits
- For the `nix-eval.yml` `@v4` choice: `actions/checkout@v5` exists and is current. `@v4` works, but if the rest of the repo is on `@v5`, prefer that for consistency.
- `oldTag: $app.stage === "production" || $app.stage === "thdxr" ? "" : "v1"` should arguably also become a constant once this lands — the conditional made sense pre-rollout, post-rollout it's noise.

**Verdict: request-changes** (split DO migration tag flip into its own PR)
