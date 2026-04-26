---
pr: 14126
repo: All-Hands-AI/OpenHands
sha: 8eb9897e2baa0a2734e4c022d83623b3fe7f7457
verdict: request-changes
date: 2026-04-26
---

# All-Hands-AI/OpenHands #14126 — feat(settings): use from_persisted for stored settings

- **Author**: neubig
- **Head SHA**: 8eb9897e2baa0a2734e4c022d83623b3fe7f7457
- **Size**: +337 / -172 across 15 files. Mix of dependency repins (poetry.lock +47/-24, uv.lock +124/-92, pyproject.toml +6/-6, pre-commit +2/-2, two `compose.yml` agent-server image bumps), enterprise routes & storage refactors to use new `_load_persisted_*` loaders, and a small `AGENTS.md` note.

## Scope

Replaces direct `AgentSettings.model_validate(...)` / `ConversationSettings.model_validate(...)` on persisted dicts with new `_load_persisted_agent_settings` / `_load_persisted_conversation_settings` helpers from `openhands.storage.data_models.settings`. These helpers are presumed to handle migration of legacy persisted shapes via the upstream SDK's `from_persisted` API. To get the new SDK API, the PR repins `openhands-sdk`, `openhands-tools`, `openhands-agent-server` from version `1.17.0` (pinned PyPI release) to a git ref `054bbd4` of `OpenHands/software-agent-sdk`. Sandbox runtime image bumped from `1.15.0-python` / `1.18.1-python` (two different prior values across files) to `054bbd4-python`.

## Specific findings

- `openhands/storage/data_models/settings.py:73-86` (only first 14 lines of the change shown) — the diff is truncated here at the helper definitions. Full review of the helper bodies is needed; from the call-site usage I infer they accept `dict | None | AgentSettings` and dispatch to `from_persisted` when given dict shapes. **The full helper body must be reviewed** to confirm migration semantics, default-handling, and exception behaviour. This review covers everything outside that file.
- `enterprise/server/routes/org_models.py:189-194, 408-414` — two call sites: `from_org` (response builder) and `OrgDefaultsSettingsResponse.from_org`. Both now route persisted dicts through the new loaders. Good consistency.
- `enterprise/storage/org_store.py:53-58` — two getters now call the persisted loaders. Single-line each, low-risk.
- `enterprise/storage/org_store.py:223-235` (`_merge_and_validate_settings`) — significantly more complex than the old version. New flow: `base_settings = _load_persisted_*(current_settings)` → `model_dump(mode='json'[, context={'expose_secrets': True}])` → `deep_merge(base, settings_diff)` → `model_validate(merged)`. Two concerns:
  1. **Secret-exposure asymmetry**: `AgentSettings` path uses `mode='json', context={'expose_secrets': True}`; `ConversationSettings` path uses `mode='json'` only. If `ConversationSettings` contains any `SecretStr` fields, they'll be dumped as masked placeholders, then `model_validate` will store the masked placeholder as the actual secret value. Either both paths need `expose_secrets: True` or there needs to be a comment explaining `ConversationSettings` has no secrets.
  2. **Round-tripping secrets through `model_dump(...) → deep_merge → model_validate`** with `expose_secrets=True` means the bare key sits in a dict in memory longer than it did before (the prior code went straight `dict → model_validate`). Whether that's an exposure risk depends on logging/serialization downstream of `merged_settings`, but it's worth a security note in the PR.
- `enterprise/storage/user_settings.py:88-95` (`to_settings`) — switches from constructing `AgentSettings.model_validate(...)` / `ConversationSettings.model_validate(...)` to passing the raw dicts directly into `Settings(...)`. This relies on `Settings` having field validators that accept either typed objects or raw dicts (via the new `_load_persisted_*` helpers, presumably). **Confirm** `Settings` model accepts dict-shaped inputs for these fields, otherwise this is a runtime regression. The PR should include a unit test that constructs `UserSettings(...).to_settings()` from a legacy persisted dict.
- `enterprise/tests/unit/test_org_store.py:138-167` — new test `test_get_org_settings_from_org_use_persisted_loaders` patches both loader functions and asserts the getters call them. This validates the wiring but NOT the behaviour: any regression in the loader bodies themselves would not be caught by this test. Need an integration-style test that asserts a known-legacy persisted dict is correctly migrated end-to-end.
- `containers/dev/compose.yml:16` and `docker-compose.yml:11` — both bump `AGENT_SERVER_IMAGE_TAG` from `1.15.0-python` to `054bbd4-python`. **The two files were previously OUT OF SYNC** (`1.15.0-python` in `containers/dev/compose.yml` while `openhands/app_server/sandbox/sandbox_spec_service.py:16` had `ghcr.io/openhands/agent-server:1.18.1-python`). Now all three converge to `054bbd4-python`. Good — but the prior drift suggests these references should be sourced from a single constant (e.g. read from `pyproject.toml` or a shared `_versions.py`) to prevent recurrence. Out of scope for this PR but worth a follow-up issue.
- `dev_config/python/.pre-commit-config.yaml:60-61` — mypy `additional_dependencies` switches from PyPI pins (`openhands-sdk==1.17.0`, `openhands-tools==1.17.0`) to git-ref installs at `054bbd4`. This is necessary so commit-time mypy doesn't fall back to the released SDK, as called out in the new `AGENTS.md` note. The git installs add real wall-clock time to first pre-commit invocation (cold cache: clone the repo, build the package). For developers iterating quickly this is a meaningful UX hit. Worth keeping until the SDK release that includes `from_persisted` is cut, then immediately revert to a versioned pin.
- `pyproject.toml`, `poetry.lock`, `uv.lock`, `enterprise/poetry.lock` — three lockfile updates plus `pyproject.toml` switching the three openhands packages to git deps at `054bbd4`. The `openhands-sdk` change cascades a `lmnr >= 0.7.47` minimum (was `>= 0.7.24`), and `lmnr` itself bumps `0.7.46 → 0.7.48`, which in turn tightens upper bounds on `packaging` (`<27.0`) and `tqdm` (`<5.0`). All three of those are tightenings, not loosenings — increases the chance of solver conflicts in downstream environments. Check that no production deployment pins `tqdm >= 5` or `packaging >= 27` via another path.
- **AGENTS.md note**: helpful, but it's documenting a workflow that should ideally not exist long-term. The note essentially says "if you pin to an unreleased SDK commit, you must also pin the pre-commit mypy deps to the same commit and update the agent-server image tag." That's three places to remember, in lockstep. A small CI check that verifies these three are in sync would prevent future drift.

## Risk

**High**. Three compounding factors:
1. **Switching from PyPI pins to git refs across three core packages** in the production dependency graph. Reproducibility now depends on `OpenHands/software-agent-sdk` not being force-pushed and on the cached wheels being available. Until the SDK ships `1.18.x` with `from_persisted`, every fresh install builds from git.
2. **Behavioural change in `_merge_and_validate_settings`** with potentially-different secret-handling semantics between `AgentSettings` and `ConversationSettings` paths.
3. **Loss of rollback granularity** — the dependency repin and the call-site refactor land together. If the new SDK has a `from_persisted` regression, revert is "drop the whole PR"; you can't keep the call-site cleanup while reverting the SDK pin.

## Verdict

**request-changes** — three asks before merge:
1. Land the SDK / tools / agent-server bump as its own PR (or at minimum its own commit) so it can be reverted independently of the call-site refactor. Current commit shape couples them.
2. Clarify the secret-handling asymmetry in `_merge_and_validate_settings`: either both paths use `expose_secrets: True` or document why `ConversationSettings` is safe without it.
3. Add an integration-style test (not just a mock-based wiring test) that takes a known legacy persisted dict for both `AgentSettings` and `ConversationSettings` and asserts the migrated result. The current `test_get_org_settings_from_org_use_persisted_loaders` only validates that the loader was called.

Optional: extract the `054bbd4-python` image tag into a single source of truth referenced from both compose files and `sandbox_spec_service.py` so the three references can't drift again.
