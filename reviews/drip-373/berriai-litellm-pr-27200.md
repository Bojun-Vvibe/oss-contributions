# BerriAI/litellm #27200 — helm: skip proxy startup prisma db push when migrations Job is enabled

- **Head:** `c9b2481` (`c9b24818e7a512cba660789c92af94f9c8f37a38`)
- **Repo:** BerriAI/litellm
- **Scope:** `deploy/charts/litellm-helm/templates/deployment.yaml:131-141` (10 inserted lines, 0 removed)

## What the diff does

When `.Values.migrationJob.enabled` is true, the deployment template now appends `DISABLE_SCHEMA_UPDATE: "true"` to the proxy container's `env:` list. Placement is intentional: the new block sits *after* both `.Values.envVars` and `.Values.extraEnvVars` rendering, so under Kubernetes' last-wins duplicate-env-name semantics it cannot be silently shadowed by a user-supplied `DISABLE_SCHEMA_UPDATE` in `extraEnvVars`. The accompanying comment at `:134-139` documents that intent inline (and references that the migrations Job already follows the same pattern).

## Why this is necessary

When `migrationJob.enabled=true`, schema ownership is supposed to be the dedicated `Job` (which runs once per release with `helm.sh/hook: pre-install,pre-upgrade`). But the proxy `Deployment` was still running `prisma db push` on every pod startup, so on a 5-replica HA rollout you'd get up to 5 concurrent `prisma db push` calls racing against the same Postgres, plus racing the migrations Job itself. Symptoms in the wild are intermittent `prisma migration_lock` advisory-lock contention errors during rollouts.

## Strengths

- **Placement-after-extraEnvVars is the load-bearing detail** and is correctly called out in the inline comment. Without that ordering rule, a chart user could (intentionally or accidentally) override the safety with a `DISABLE_SCHEMA_UPDATE: "false"` in `extraEnvVars`.
- **Conditional gate on `migrationJob.enabled`** is the right toggle key — it inherits the existing chart contract that "enable the Job ⇔ Job owns schema" without introducing a new value.
- **Strictly additive, no template changes elsewhere.** Default `migrationJob.enabled=false` users see byte-identical rendered manifests.

## Concerns

1. **No values-doc update visible in this slice.** `values.yaml` should have a comment under `migrationJob.enabled` noting the side-effect that `DISABLE_SCHEMA_UPDATE=true` will be force-set on the proxy when this is true. Without that, a chart user who later debugs "why isn't my proxy running migrations" has to read template internals.
2. **No test in `tests/` for the chart.** The litellm-helm chart has a `tests/` directory with `helm unittest`-style coverage; this PR should add a case asserting the env var is rendered when `migrationJob.enabled=true` and absent when `false`. Easy to add and prevents the next refactor from regressing the placement-after-extraEnvVars invariant.
3. **No mention of `initContainers` migration pattern.** Some operators use an init-container to run migrations instead of the chart's `Job`. This PR's gate is specifically `migrationJob.enabled`, so init-container users will not get the same benefit. Out of scope for this PR but worth a follow-up issue.
4. **Behavior on rollback.** If a user toggles `migrationJob.enabled` from true → false on a rollback, the proxy will resume running `prisma db push` on startup against a schema the Job left at version N. As long as `prisma db push` is idempotent against current schema (it is), this is fine; but worth a one-line note in the PR description.

## Verdict

**merge-after-nits** — add a `values.yaml` comment + one `helm unittest` case before merge. The change itself is correct, minimal, and addresses a real HA-rollout race; head `c9b2481`.
