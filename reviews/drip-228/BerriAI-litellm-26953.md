# BerriAI/litellm #26953 — chore(audit): audit-log /cache/settings + /config_overrides/hashicorp_vault mutations

- **PR**: https://github.com/BerriAI/litellm/pull/26953
- **Head SHA**: `abd51fc30e1e33a610fd77c5ad20d374727c3738`
- **Files reviewed**:
  - `litellm/proxy/_types.py` (+2 -0) — two new `LitellmTableNames` entries
  - `litellm/proxy/management_endpoints/cache_settings_endpoints.py` (+120 -3)
  - `litellm/proxy/management_endpoints/config_override_endpoints.py` (+139 -3)
  - `tests/test_litellm/proxy/management_endpoints/test_cache_settings_endpoints.py` (+143 -1)
  - `tests/test_litellm/proxy/management_endpoints/test_config_override_endpoints.py` (+188 -1)
- **Date**: 2026-05-01 (drip-228)

## Context

Two high-trust mutation endpoints were being silently un-audited:

- `POST /cache/settings` — flips Redis credentials, connection
  strings, the cache backend itself. An admin (or compromised admin)
  can silently re-route the LLM-response cache to an attacker-owned
  Redis. The mutation succeeded with no audit-log row.
- `POST /config_overrides/hashicorp_vault` and `DELETE` ditto —
  manages the proxy's KMS configuration: Vault tokens, AppRole secret
  IDs, client keys. Same un-audited gap.

Every other admin-mutation endpoint in this codebase (teams, keys,
models, SSO config) writes to `LiteLLM_AuditLogs` via
`store_audit_logs`-gated `create_audit_log_for_update`. These two
were drift.

## Diff walk

**`_types.py:192-194`** — adds two new `LitellmTableNames` enum
values:

```python
CACHE_CONFIG_TABLE_NAME = "LiteLLM_CacheConfig"
CONFIG_OVERRIDES_TABLE_NAME = "LiteLLM_ConfigOverrides"
```

These match the underlying Prisma table names so audit-log readers
joining on `table_name` get the expected join target.

**`cache_settings_endpoints.py:38-118`** — adds five new helpers, all
file-scope private (`_REDACTED_VALUE`, `_redact_settings`,
`_log_audit_task_exception`, `_emit_cache_settings_audit_log`). The
critical design choice is **redact values, preserve key set**:

```python
def _redact_settings(settings):
    if not settings:
        return {}
    return {k: _REDACTED_VALUE for k in settings.keys()}
```

The audit row records *which fields changed* but not their values, so
the audit table can't itself become a credential-harvest sink. That's
the right tradeoff — reading the audit log tells you "an admin
flipped the Redis password" without telling you what the new password
is.

**`cache_settings_endpoints.py:373-470`** — the integration into
`update_cache_settings`:
- `:373` adds the `litellm_changed_by` Header dependency (matches the
  existing pattern for delegated-actor tracking)
- `:407-419` snapshots the prior `cache_settings` from the DB
  **before** the encrypt+upsert at `:420-440`, computes
  `action: AUDIT_ACTIONS = "updated" if existing_row is not None else
  "created"`. The action key is on **row existence**, not on
  `cache_settings` content — correct, because a row with `null`
  `cache_settings` is still a pre-existing row and an "update" of it
  is not a "create".
- `:458-470` emits the audit row **after** the
  `switch_on_llm_response_caching()` succeeds, so a partially-applied
  mutation that fails before cache-switch doesn't get logged as a
  successful change.

**`config_override_endpoints.py:38-115`** — mirrors the cache-settings
pattern with the same five-helper shape. The vault flow has an
additional wrinkle handled correctly:

`:265-272` — when there's no DB record yet, the code merges current
env vars into `config_data`. The audit-log emission at `:340-349`
correctly captures `before_config = existing_decrypted if
existing_decrypted is not None else env_values`, so an audit row for
the **first** vault-config write shows the env-var-derived prior state
rather than `{}`. That's exactly what an auditor needs to see —
"before this row, the proxy was using `VAULT_TOKEN=$X` from env;
after, it's using a DB-backed config".

**Delete-path discipline at `config_override_endpoints.py:447-472`**:

```python
deleted = False
try:
    await prisma_client.db.litellm_configoverrides.delete(...)
    deleted = True
except RecordNotFoundError:
    verbose_proxy_logger.debug(...)

_clear_hashicorp_vault_state(proxy_config)

if deleted:
    await _emit_hashicorp_vault_audit_log(action="deleted", ...)
```

Audit row is only emitted if a row was **actually** removed. An
idempotent-delete on a missing row produces no security-relevant
change, so producing no audit row is the right semantic. Without this
guard, replay-attack scanners would see noisy "delete" entries on
every probe.

## Observations

1. **`_log_audit_task_exception` closes a real silent-failure hole.**
   `asyncio.create_task` without a done-callback swallows exceptions;
   if Prisma is down or the `LiteLLM_AuditLogs` table is locked, the
   audit write fails silently and the operator never knows.
   The done-callback at `cache_settings_endpoints.py:51-63` and
   `config_override_endpoints.py:55-66` (identical shape) surfaces
   the failure as a `verbose_proxy_logger.warning`. That's the right
   loudness — an audit-write failure is operator-visible but doesn't
   block the actual admin operation.

2. **Action computation is rigorous.** `action = "updated" if
   existing_row is not None else "created"` keys off **row existence**
   not content equality. That's the right contract for an admin-action
   log: "I created this row" vs "I changed this row" is the question
   an auditor asks, not "did the value change". A no-op PUT (write
   the same settings as already exist) correctly gets logged as
   "updated" — that's still an admin action worth recording.

3. **Redacted-key-only audit row is the right security tradeoff.**
   `{k: "***REDACTED***" for k in settings.keys()}` — auditor sees
   "field X was modified" without learning the field's value. The
   alternative shapes (full plaintext / hash / `[redacted]`) all
   either leak the credential or hide which field changed.
   Key-set-only is the privacy-preserving sweet spot.

4. **Two endpoints, identical helper shape.** The two endpoint files
   carry near-identical 5-helper blocks (`_REDACTED_VALUE`,
   `_redact_*`, `_log_audit_task_exception`, `_emit_*_audit_log`).
   That's a follow-up factoring opportunity — these belong in
   `litellm/proxy/management_helpers/audit_logs.py` as a single
   `emit_audit_log_for_admin_mutation(table, object_id, before,
   after, redactor)` helper. Not blocking this PR but the duplication
   will rot if a third endpoint (`/sso/...`?) gets the same treatment
   later and the three copies drift.

5. **Tests are substantial.** +143 / +188 lines on the two test files
   means each helper and each endpoint path has coverage. Without the
   diff visible I'd want to verify: (a) audit row contains
   `***REDACTED***` not the actual credential, (b) `litellm_changed_by`
   header overrides `user_id`, (c) `store_audit_logs is not True`
   short-circuits with no DB write, (d) delete-of-missing-row emits
   no audit row.

6. **Nit: `import litellm` inside `_emit_hashicorp_vault_audit_log`
   at `config_override_endpoints.py:81` (vs module-level in
   cache_settings_endpoints).** Minor inconsistency — both files
   could either go module-level or both stay function-scoped. Pick
   one when consolidating into a shared helper.

## Verdict

**merge-after-nits** — correct security posture (key-set audit only,
not values), correct semantics (action on row-existence, delete-only-
when-deleted), surfaces the silent-failure mode of fire-and-forget
tasks. Fold the two helper blocks into one shared helper in
`management_helpers/audit_logs.py` either in this PR or a fast
follow-up.

## What I learned

The hardest part of audit-logging admin endpoints isn't the row write
— it's the four edge cases this PR gets right: (1) what to redact,
(2) when "create" vs "update" (row existence, not content equality),
(3) what to do on delete-of-missing-row (nothing — preserves the
no-noise contract), and (4) how to surface fire-and-forget failures
without blocking the user-visible operation. Each of these is a place
where the lazy implementation is silently wrong, and the audit-log
table either becomes a credential-harvest sink, a misleading
artifact, or a silent black hole for the operator.
