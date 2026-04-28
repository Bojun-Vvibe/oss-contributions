# BerriAI/litellm #26643 — [Fix] /config/update: targeted per-section writes, drop store_model_in_db gate

- **PR**: [BerriAI/litellm#26643](https://github.com/BerriAI/litellm/pull/26643)
- **Head SHA**: `abbe5d7f`

## Summary

Rewrites the `/config/update` admin endpoint to perform *targeted per-section
writes* (one Prisma `litellm_config` row per section the request actually sent)
instead of the prior pattern of `proxy_config.get_config()` (which loads the
*entire* on-disk YAML + DB-merged view), mutating only the requested sections,
then `proxy_config.save_config(new_config)` (which serializes the *whole
merged view* back to DB). The old shape persisted pre-existing YAML values
to DB as a side effect of any unrelated update, which is the bug the PR title
calls out. Also drops the blanket `if store_model_in_db is not True: raise 500`
guard at the top — the endpoint now works on installs that don't have model-in-DB
mode enabled, since none of its sections actually touch the model store.

## Specific findings

- `litellm/proxy/proxy_server.py:12557-12643` — endpoint body fully rewritten.
  Two helpers introduced inline: `_read_section(param_name) -> dict` does a
  `find_first` against `litellm_config` and returns `dict(row.param_value)` or
  `{}` if missing; `_upsert_section(param_name, value)` does a `json.dumps`
  + `upsert` with both `create` and `update` payloads. Both helpers are
  scoped to the endpoint (closures), which is the right call — they're not
  reusable surface, and exposing them at module level would invite consumers
  that don't go through the section-merge logic above.
- The blanket `if store_model_in_db is not True: raise HTTPException(500, ...)`
  guard is gone. Correct — none of the four sections this endpoint writes
  (`general_settings`, `environment_variables`, `litellm_settings`,
  `router_settings`) are part of the model-store, so the guard was
  conflating "we're using DB-stored config" with "we have any DB at all".
  The endpoint still requires `prisma_client is not None` so installs without
  any DB are still rejected.
- `general_settings` block at `:12591-12604` — keeps the
  `alert_to_webhook_url` side effect of auto-enabling slack alerting, but
  now the side effect is applied to the *DB-loaded* `existing` dict not a
  cross-source merged view. Subtle behavior change: prior code's
  `_existing_settings = config["general_settings"]` came from the merged
  YAML+DB view, so YAML-only `alerting` entries influenced the auto-enable
  logic; now only DB-side `alerting` does. For installs that bootstrap
  alerting from YAML and then update via UI, this means the slack side
  effect could redundantly add `"slack"` to a list that already had it via
  YAML — the `if "slack" not in existing["alerting"]` guard handles the
  in-DB dedup but nothing dedupes against YAML. Likely fine since the
  effective config view is the union, but worth a comment naming the
  YAML-blindness.
- `environment_variables` at `:12605-12609` — encrypts each value via
  `encrypt_value_helper` before writing, then merges into `existing` (DB
  values are kept unless the request explicitly overrides). Correct shape —
  prior code did the same encryption but against the merged view.
- `litellm_settings` at `:12611-12628` — preserves the legacy
  *existing-wins* merge (`{**updated, **existing}`) which is the
  load-bearing quirk of this section: it's intentional that admin-UI
  updates *don't* clobber pre-existing DB settings (only the explicit
  `success_callback` union path is allowed to grow). The docstring update
  at `:12559-12565` correctly names this as the "Sections the caller did
  not send are left untouched" invariant. The `success_callback` union
  uses `normalize_callback_names` + `set(...)` for dedup — same as before.
- `router_settings` at `:12630-12634` — *request-wins* merge
  (`{**existing, **updates}`), opposite of `litellm_settings`. Both shapes
  exist for legitimate reasons (router settings are tuning knobs admin
  wants to drive from UI; litellm settings include callback config that
  YAML deliberately seeds and shouldn't get clobbered) but the asymmetry
  is the kind of thing that bites a future engineer rewriting the merge
  logic. Worth one comment per branch naming the policy.
- Test rewrite at `tests/proxy_unit_tests/test_proxy_server.py:2764-2815` —
  drops the `store_model_in_db = True` setattr (no longer required), drops
  the `MockProxyConfig.get_config` / `save_config` machinery (no longer
  called), wires a `FakeRow` + `fake_find_first` + `fake_upsert` shape
  capturing the upsert kwargs into a dict, and the assertion at `:2810`
  checks `upserted["litellm_settings"]["success_callback"]` directly
  rather than reaching through the `mock_proxy_config.saved_config` chain.
  Sharper assertion — pins the section-targeted invariant by structural
  shape (only the modified section appears in `upserted`, not all four).
- New test file `tests/test_litellm/proxy/management_endpoints/test_update_config_endpoint.py`
  introduces a `FakeLitellmConfig` reusable fixture (init from a row dict,
  exposes `find_first` + `upsert` AsyncMocks, records `upsert_calls`).
  The full file is 241 lines per the diff hunk; the structural pattern
  here lets each test assert "section X was upserted with shape Y *and*
  section Z was *not* touched" which is exactly the right invariant for
  the new targeted-write contract.

## Nits / what's missing

- The `litellm_settings` merge policy (existing-wins) and `router_settings`
  policy (request-wins) need one-line comments naming why they differ. A
  future "let's make these consistent" refactor will silently change
  behavior otherwise.
- `_upsert_section` does `json.dumps(value)` and passes the string to both
  `create` and `update`. Prisma's `Json` column accepts both raw dicts and
  JSON strings depending on the driver version; the prior path used
  `prisma_client.jsonify_object` (which the new code drops). Worth a
  one-line confirmation in the test that round-tripping a dict-with-nested-
  list survives the `json.dumps → DB → param_value` path with the actual
  shape (`tests/test_litellm/...test_update_config_endpoint.py` likely
  covers this in its longer body but the diff truncates).
- `general_settings` `alert_to_webhook_url` side effect uses `existing["alerting"]`
  as a list when the YAML-bootstrapped version might be a string. The
  `isinstance(existing["alerting"], list)` guard at `:12598` handles the
  read side but the `existing["alerting"] = ["slack"]` write at `:12595`
  unconditionally writes a list, which would clobber a string-typed
  `alerting` value. Edge case but documented as legitimate config shape
  in litellm.
- Removing the `store_model_in_db` guard is a behavior change worth a
  release-notes line — installs that previously got a 500 on this endpoint
  will now get a 200, which is the intended fix but observable.

## Verdict

**merge-after-nits** — the targeted-write rewrite is the right architectural
fix for the side-effect bug (per-section upserts make "what gets written" a
function of "what was sent" not "what the merged view happens to contain"),
the test rewrite sharpens assertions to pin the new contract structurally,
and dropping the `store_model_in_db` blanket gate correctly scopes the
guard to actual model-store operations. Nits (merge-policy comments,
`alerting`-as-string edge case, release-notes mention of the gate removal)
are real but small enough to land as inline tweaks.
