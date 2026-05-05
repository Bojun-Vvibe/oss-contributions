# BerriAI/litellm PR #27160 — [Fix] Proxy: Break managed-resources import cycle on Python 3.13

- Repo: `BerriAI/litellm`
- PR: #27160
- Head SHA: `0976fbc6c408`
- Author: `yuneng-berri`
- Scope: 9 files, +112 / -30
- Verdict: **merge-after-nits**

## What it does

Two unrelated cleanups bundled in one PR:

### 1. Real fix — break a circular import that bites on Python 3.13

- Adds `user_api_key_has_admin_view(user_api_key_dict)` helper directly in `litellm/proxy/_types.py:2738-2748` (returns `True` for `PROXY_ADMIN` / `PROXY_ADMIN_VIEW_ONLY`).
- Deletes the duplicate definition from `litellm/proxy/management_endpoints/common_utils.py:27-32` and re-exports the new name from `_types` for backward compat (`from litellm.proxy._types import user_api_key_has_admin_view as _user_has_admin_view  # noqa: F401  re-exported`).
- Updates the leaf consumer at `litellm/llms/base_llm/managed_resources/isolation.py:14-17` to import from `_types` instead of `proxy.management_endpoints.common_utils` — which previously dragged in `litellm.proxy.utils` transitively and created the cycle.
- Reorders `litellm/proxy/hooks/__init__.py` so that `PROXY_HOOKS` and `get_proxy_hook` are **defined first**, and only after that does the module attempt `from enterprise.enterprise_hooks import ENTERPRISE_PROXY_HOOKS`. The block-comment at `hooks/__init__.py:14-17` correctly explains the intent: the enterprise package re-imports back into `proxy.hooks` to look up hook classes, and on Py 3.13 the partially-initialized-module behaviour is stricter, so the symbols have to exist before the back-import fires.

### 2. CI / OTEL test plumbing

- `litellm/proxy/example_config_yaml/otel_test_config.yaml:48-50` adds `require_auth_for_metrics_endpoint: False` so the `/metrics`-scrapes-without-credentials test (`tests/otel_tests/test_prometheus.py`) keeps working after a default-secure flip elsewhere.
- `.circleci/config.yml:1478` drops `-x` from `pytest -v tests/otel_tests` so a single-test failure no longer kills the rest of the suite.
- `.circleci/config.yml:1938` adds `NODE_OPTIONS=--experimental-vm-modules` to the Vertex/AI Studio Jest run for ESM-mocking compatibility.
- `tests/otel_tests/test_key_logging_callbacks.py:67` flips an assertion string from `"Failed to load vertex credentials"` → `"GCS_BUCKET_NAME is not set in the environment"`, presumably to match the message produced after the new config defaults; this is a behaviour-locking test, so it should be matching whatever the production error path now emits.

## What's good

- **Correct cycle-break.** Pulling the role-check helper down to `_types` (a low-level module already imported by everything in the proxy stack) is the right place. `_types` has no upward imports into `proxy.utils`, so any leaf module can safely call it.
- **Explicit re-export with `noqa: F401`** preserves any external code that imports `_user_has_admin_view` from `common_utils` — semver-friendly.
- **Block comment in `hooks/__init__.py:14-17` actually explains why** the import ordering matters, which is the kind of comment that survives the next refactor and prevents the cycle from re-introducing itself.
- **Hooks test config change is documented inline** with a comment explaining why `/metrics` is opted out of auth in tests.

## Issues / nits

1. **PR is two changes in a trench-coat.** The import-cycle fix is one thing; the CircleCI/Jest/OTEL test fixture changes are another. They should ideally be split: a future bisect will land on a 9-file commit and have to figure out whether the import or the test config caused the regression.

2. **Asserting "GCS_BUCKET_NAME is not set" looks like it papers over a regression.** The previous assertion was `"Failed to load vertex credentials"` — a Vertex-specific error. The new one is GCS-specific. Either:
   - the underlying default callback fixture changed and the test should be following that change (then a one-line note in the PR body would help reviewers); or
   - the auth/init order changed and the test is now hitting the GCS check before the Vertex check (a real behaviour change that deserves its own discussion).
   The PR body doesn't mention this — worth a sentence.

3. **Removing `-x` from the OTEL pytest run** is fine for *getting more signal per CI run*, but it also means a previously-fail-fast suite now runs to completion. If any of those OTEL tests have side-effects on a shared collector/server, ordering bugs could surface. Not blocking, but worth a note.

4. **`# noqa: F401  re-exported`** at `common_utils.py:19` works, but a more discoverable pattern is `__all__ = [...]` plus the import — that way grep/IDE "find usages" sees the re-export intent. Optional.

5. **No new tests for the cycle fix itself.** A 1-line `tests/test_litellm/proxy/test_import_order.py` that does `python -c "import litellm.proxy._types; import litellm.llms.base_llm.managed_resources.isolation; import litellm.proxy.hooks"` and asserts no `ImportError` would lock the fix in for Py 3.13. Without it, a future reorder of the `enterprise_hooks` import in `hooks/__init__.py` re-breaks silently outside CI.

## Verdict
**merge-after-nits** — the import-cycle fix is correct and well-explained. The bundled CI/OTEL fixture churn is fine but should ideally be its own PR, and the test-message assertion swap deserves a sentence in the PR body. A regression test for "all proxy modules import cleanly on 3.13" would be the cheapest follow-up.
