# BerriAI/litellm #27006 — fix(proxy): respect store_model_in_db=False during periodic DB model sync

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27006
- **HEAD SHA:** `84c3e6e8f59396a10fc149ed0ab4c24050957245`
- **Author:** AriOliv
- **Verdict:** `merge-after-nits`

## What the diff does

Single-file fix at `litellm/proxy/proxy_server.py:4964-4997` adding
a `store_model_in_db` precondition gate before the periodic
`add_deployment` task reads models from the DB and overwrites the
in-memory router model list:

```python
store_model_in_db_enabled = (
    general_settings.get("store_model_in_db", False) is True
    or store_model_in_db
)
if store_model_in_db_enabled and self._should_load_db_object(
    object_type="models"
):
    new_models = await self._get_models_from_db(prisma_client=prisma_client)
    # update llm router
```

Previously the gate was only `_should_load_db_object("models")`,
which defaults to True. So an operator running with
`store_model_in_db: False` (config.yaml is the source of truth)
who ever had a stray row land in `LiteLLM_ProxyModelTable` (UI
add-from-prior-config-state, manual SQL, dev environment cross-
contamination) would, on every periodic sync tick, see that row
get pulled into the in-memory router and *replace* the config.yaml
deployments — silently breaking model availability without any
log signal.

The fix reads both forms of the flag:
- `general_settings.get("store_model_in_db", False) is True` —
  the config.yaml form
- `or store_model_in_db` — the module-level global set by the
  `STORE_MODEL_IN_DB` env var

`global` declaration extended at `:4967` to add `store_model_in_db`.

Test file `tests/test_litellm/proxy/test_add_deployment_store_model_in_db.py`
(new, 210 lines) has three pinning tests:

1. `test_skips_db_model_load_when_store_model_in_db_false` —
   `general_settings={"store_model_in_db": False}`, module global
   `False`; asserts `_get_models_from_db` and `_update_llm_router`
   are never called.
2. `test_loads_db_models_when_store_model_in_db_true` —
   `general_settings={"store_model_in_db": True}`, module global
   `True`; asserts both are called once.
3. `test_loads_db_models_when_module_global_true` —
   `general_settings={}` (empty), module global `True` (env-var
   path); asserts the env-only configuration still triggers the DB
   sync, locking the OR-arm of the precondition.

## Why the change is right

The `store_model_in_db` flag's documented semantics ("Allow saving
/ managing models in DB") establish DB as either the source of
truth (True) or not (False). The previous `add_deployment` code
ignored the flag entirely on the read path, breaking the
documented invariant on the False side — config.yaml was supposed
to be authoritative but a periodic sync would silently un-
authoritative it.

Putting the gate at `add_deployment` (the periodic sync entry
point) is the right call-graph position: it's the only path where
DB → in-memory-router replacement happens, and gating here covers
every periodic-tick caller without having to find each invocation
site.

The dual-form check (`general_settings.get(...)` OR module
global) is the load-bearing detail that matches every other place
in `proxy_server.py` that reads this flag — env-var-only operators
would otherwise hit the `general_settings.get(... False)` path
returning False and be silently in the new "no sync" state when
they actually wanted sync. Test #3 explicitly pins this arm.

The `global` declaration at `:4967` extension is necessary
mechanical correctness — without it, `store_model_in_db` inside
the function body would be a NameError or a local lookup against
an unbound name.

## Nits (non-blocking)

1. **The `is True` comparison at `:4990`** is defensive but
   inconsistent with how the same flag is read elsewhere in the
   file. A simple `general_settings.get("store_model_in_db",
   False)` would handle the truthy case. The `is True` makes
   `"true"` (string), `1` (int), and any other truthy non-bool
   evaluate False — which may or may not be desired. Worth
   confirming the YAML parser always produces a Python `bool` for
   this key (it should, but operators sometimes hand-edit).

2. **No log signal when sync is skipped.** A user who changes
   `store_model_in_db: True → False` and then wonders why their
   newly-added DB model isn't appearing has no log breadcrumb.
   A `verbose_logger.debug("Skipping periodic DB model sync:
   store_model_in_db is False")` inside the `else` branch (or as
   a one-time INFO log on the first skipped sync) would close
   the diagnostic gap.

3. **`_init_non_llm_objects_in_db` is unaffected** — the test
   patches it to AsyncMock but doesn't assert on it. Worth a
   one-line confirmation in the PR description that this fix is
   intentionally scoped to *models* and that other DB-backed
   objects (teams, keys, budgets) continue to sync regardless of
   `store_model_in_db`. Otherwise a reader could mistakenly
   conclude that all DB sync is now gated by this flag.

4. **The fix doesn't address pre-existing stale router state.**
   If an operator has been running with `store_model_in_db: False`
   and the DB has been silently overwriting their in-memory
   router for some time, this PR stops the bleeding but doesn't
   restore the config.yaml models. A note in the release-notes /
   PR description recommending a router restart after upgrade
   would help operators close the gap.

5. **Comment at `:4983-4988` is excellent.** "Otherwise a stale
   row in LiteLLM_ProxyModelTable can silently replace models
   defined in config.yaml on every periodic sync" is exactly the
   failure mode in plain English and should stay verbatim.

## Verdict rationale

Right gate at the right call-graph position (the periodic sync
entry point), correct dual-form flag read matching the rest of
the file, locking tests for all three states (False/True/env-only),
and a load-bearing comment that names the exact failure mode. The
`is True` strictness, missing log breadcrumb, and post-upgrade
recovery guidance are polish nits.

`merge-after-nits`
