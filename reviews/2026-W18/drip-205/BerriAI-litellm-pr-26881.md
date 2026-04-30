# Review: BerriAI/litellm#26881 — feat(helm): migration job annotation control

- PR: https://github.com/BerriAI/litellm/pull/26881
- Author: beli-sk (Michal Belica)
- headRefOid: `9a1da12776c10d94fa32827881b5b8b27f839920`
- Files: `deploy/charts/litellm-helm/templates/migrations-job.yaml` (+7/-4),
  `deploy/charts/litellm-helm/values.yaml` (+9/-0),
  `deploy/charts/litellm-helm/tests/migrations-job_tests.yaml` (+32/-0)
- Verdict: **merge-after-nits**

## Analysis

Adds three optional Helm-chart knobs on the migrations Job: a free-form `migrationJob.jobAnnotations`
map, plus `hooks.argocd.{hook,hookDeletePolicy}` and `hooks.helm.{hook,hookDeletePolicy}` overrides
that previously had baked-in `PreSync` / `BeforeHookCreation` and `pre-install,pre-upgrade` /
`before-hook-creation` literals. The template diff at `templates/migrations-job.yaml:9-19` correctly
wraps the user map first with `{{- with .Values.migrationJob.jobAnnotations }}{{- toYaml . | nindent 4 }}{{- end }}`,
then renders the hook annotations with `| default "..." | quote` so existing installs without
overrides keep their current behavior.

The Helm test fixture at `tests/migrations-job_tests.yaml:257-288` exercises every new path: setting
`jobAnnotations.test: test1` and overriding all four hook strings, then asserting each annotation
key gets the override. That's good test discipline for a values-only change.

**Nit 1 (worth fixing before merge):** `values.yaml:329-340` shows the new hook fields commented
out with `# hook: PreSync` / `# hookDeletePolicy: BeforeHookCreation`. The hooks block is also
listed *below* the new `jobAnnotations: {}`, but the block-level comment "extra annotations for
pods created from the Job" was clobbered/repositioned (the pre-existing `annotations:` was for
pod-level annotations on the Job's pod template). Re-reading the diff: the old comment wording
"extra annotations for pods created from the Job" actually got attached to a different field after
the insertion. Worth a quick re-read of the final values.yaml to make sure `annotations:` (pod
template) and `jobAnnotations:` (Job metadata) are clearly distinguished in comments — these two
keys are easy to confuse.

**Nit 2 (non-blocking):** `helm.sh/hook-weight` is still hard-coded to `{{ .Values.migrationJob.hooks.helm.weight | default "1" | quote }}` (already configurable, good). For symmetry with the new ArgoCD hook overrides, consider whether `argocd.argoproj.io/sync-wave` should also be exposed — but this is scope creep for the bug being fixed (#26875).

The change is correctly backward compatible (defaults preserve current behavior), well-tested for a
chart change, and the only real risk is the values.yaml comment placement. Merge after a quick
comment cleanup, otherwise it's a clean infra PR.
