# BerriAI/litellm #27176 — [Fix] Helm: honor external DB secret in standalone mode

- Head SHA: `40623d956137a83fff08cf4068a0ff12b79be6fd`
- Author: ben-wangz
- Diff: +50 / −7 across 4 chart files (`_helpers.tpl`, `deployment.yaml`, `migrations-job.yaml`, `secret-dbcredentials.yaml`)
- Closes: #27173 ; related: #13544, #21716

## Verdict: `merge-after-nits`

## What it does

Fixes a real Prisma `P1000` startup failure in the helm chart's standalone DB mode. Before this PR:
- The deployment + migrations job hardcoded the secret name `<fullname>-dbcredentials` and pulled `username`/`password` from chart-managed values — so even when an operator set `db.dbCredentialsSecretName` or `postgresql.auth.existingSecret` to point at an external secret, the workload still read the chart-rendered fallback secret.
- The fallback secret was always rendered when `db.deployStandalone=true`, regardless of whether an external secret was configured — risking credential drift if both existed.

The fix introduces two helper templates and one rendering-gate change:

1. **`litellm.dbCredentialsSecretName`** (`_helpers.tpl:107-115`) — explicit precedence: `.Values.db.dbCredentialsSecretName` → `.Values.postgresql.auth.existingSecret` → `<fullname>-dbcredentials`. Documented in the template comment.
2. **`litellm.shouldCreateDbCredentialsSecret`** (`_helpers.tpl:121-127`) — returns the string `"true"` only when neither external secret value is set; the fallback `secret-dbcredentials.yaml` resource is now gated on this (`secret-dbcredentials.yaml:1`), so it stops being created when an external secret is configured.
3. **deployment.yaml** (`deployment.yaml:65-72`) — `DATABASE_USERNAME` / `DATABASE_PASSWORD` now read from `{{ include "litellm.dbCredentialsSecretName" . }}` with configurable `key:` (`.Values.db.secret.usernameKey | default "username"`).
4. **migrations-job.yaml** (`migrations-job.yaml:75-91`) — adds the same `DATABASE_USERNAME` / `DATABASE_PASSWORD` / `DATABASE_HOST` / `DATABASE_NAME` env block to the migration job, replacing the previously-inlined `postgresql://user:pass@host/db` URL.

Also: the `secret-dbcredentials.yaml` file gets a trailing-newline fix (the `\ No newline at end of file` marker in the diff).

## What's good

1. **Precedence is unambiguous and documented** in the template header, matches the PR description, and is intentionally backward-compatible: existing deployments with neither value set continue to land on the legacy `<fullname>-dbcredentials` name, so this is a non-breaking change for default installs.
2. **String-equality gate, not boolean**: `eq (include "litellm.shouldCreateDbCredentialsSecret" .) "true"` (`secret-dbcredentials.yaml:1`) is the correct sprig idiom — `include` always returns a string, so a naive `if (include "...")` would treat any non-empty output (including the literal `"false"`) as truthy. The author got this right.
3. **Configurable secret keys via `db.secret.usernameKey` / `db.secret.passwordKey`** with sane defaults — this lets external secrets like AWS Secrets Manager mirrors (which often use `DB_USER`/`DB_PASS`) work without renaming.
4. **Validation evidence is concrete**: rendered templates inspected, ArgoCD redeploy on clean k3s, no `P1000` after the fix, and no `litellm-dbcredentials` resource created when external secret is configured. That's the right shape of validation for a chart change that has no unit-test harness.
5. **Migration job now uses the same secret as the deployment** instead of inlining cleartext password into a `DATABASE_URL` env var — an actual security improvement on top of the bugfix. Previously the password was b64-encoded into the rendered manifest visible via `helm template`; now it's pulled from a Secret reference at pod start.

## Nits worth raising before merge

### 1. `migrations-job.yaml` has a buggy fallback path: `value: {{ .Values.db.url | quote }}` is rendered *unconditionally* in the standalone branch

Look at the diff at `migrations-job.yaml:88-91`:
```yaml
{{- else if .Values.db.deployStandalone }}
- name: DATABASE_USERNAME
  valueFrom:
    secretKeyRef: ...
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef: ...
- name: DATABASE_HOST
  value: {{ .Release.Name }}-postgresql
- name: DATABASE_NAME
  value: {{ .Values.postgresql.auth.database | default "litellm" }}
- name: DATABASE_URL
  value: {{ .Values.db.url | quote }}
{{- end }}
```

The `DATABASE_URL` line at the end is now `{{ .Values.db.url | quote }}` — but this is the **`else if .Values.db.deployStandalone` branch**, which by construction means `.Values.db.url` is *not* set (otherwise the outer `if .Values.db.url` would have matched). So `DATABASE_URL` will be the empty string `""`. The deployment.yaml side doesn't set `DATABASE_URL` in this branch at all and lets litellm's startup code assemble it from the four parts — but the migration job *does* set it to empty string, which downstream may interpret as "URL configured but blank" and fail in a confusing way.

Either delete the `DATABASE_URL` line from the standalone branch in `migrations-job.yaml` (mirror what deployment.yaml does) OR construct it from the four parts the way the previous code did but with the username/password fetched via `valueFrom` (which isn't trivial in env-var assembly — you'd need an init container or a `command:` that builds the URL from env). The safest fix is to delete the line and trust litellm's existing logic to assemble from `DATABASE_HOST`/`DATABASE_NAME`/`DATABASE_USERNAME`/`DATABASE_PASSWORD`, mirroring the deployment.

This is the only behavioural concern I'd want addressed before merge.

### 2. `db.secret.usernameKey` / `db.secret.passwordKey` are new values but `values.yaml` doesn't appear in the diff

The `default "username"` / `default "password"` makes this safe at runtime, but the chart's `values.yaml` should declare these new keys (with comments) so they're discoverable via `helm show values`. Otherwise an operator using an external secret with `DB_USER`/`DB_PASS` keys won't know the override exists without reading the template source.

### 3. `litellm.dbCredentialsSecretName` falls back to the legacy default but deployment.yaml's existing `DATABASE_HOST` line is unchanged

`deployment.yaml:74-75` still hardcodes `value: {{ .Release.Name }}-postgresql` for `DATABASE_HOST`, which is the chart-managed postgres subchart. If an operator sets `db.dbCredentialsSecretName` to an *external* DB's secret (i.e. they're not using the bundled postgres), the username/password come from the external secret but `DATABASE_HOST` still points at the (probably non-existent) chart-managed postgres. The PR scope is "secret name resolution" so this isn't a regression introduced here, but it's a logical follow-up: if you're consuming an external secret, you almost certainly need an external host too. Worth a `db.dbHost` override sibling in a follow-up PR or at least a `NOTES.txt` warning.

### 4. Trailing-newline change is unrelated but harmless

`secret-dbcredentials.yaml` now ends with a newline; previously it didn't. Bundling unrelated whitespace fixes is a minor style point, not a blocker.

## Why not request-changes

The core bug (chart ignores external secret config) is real and well-targeted. The fix has the right precedence semantics, the right `include`-string-comparison gate, and the right backward-compatibility story. The `DATABASE_URL` empty-string in the migration job is the one real concern but it's a one-line delete to fix.

## Citations

- `deploy/charts/litellm-helm/templates/_helpers.tpl:99-127` — two new helper templates
- `deploy/charts/litellm-helm/templates/deployment.yaml:65-72` — secret reference + configurable key
- `deploy/charts/litellm-helm/templates/migrations-job.yaml:75-91` — env block change (and the suspect `DATABASE_URL` line)
- `deploy/charts/litellm-helm/templates/secret-dbcredentials.yaml:1` — gating change
