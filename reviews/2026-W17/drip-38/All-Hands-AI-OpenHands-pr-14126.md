# All-Hands-AI/OpenHands #14126 — feat(settings): use from_persisted for stored settings

- **Repo**: All-Hands-AI/OpenHands
- **PR**: [#14126](https://github.com/All-Hands-AI/OpenHands/pull/14126)
- **Head SHA**: `8eb9897e2baa0a2734e4c022d83623b3fe7f7457`
- **Author**: neubig
- **State**: OPEN (+337 / -172)
- **Verdict**: `merge-after-nits`

## Context

The agent SDK is moving from "construct settings via
`AgentSettings.model_validate(dict)`" to a dedicated
`AgentSettings.from_persisted(dict)` loader that handles legacy
shapes, defaults, and cross-version migration. This PR routes
all enterprise settings reads through that loader and pins
OpenHands to the SDK commit (`054bbd4`) and matching agent-server
image that introduced it.

## Design

Three coordinated moves:

1. **Centralized loader use** in enterprise routes/storage.
   - `enterprise/server/routes/org_models.py:18-21` imports
     `_load_persisted_agent_settings` and
     `_load_persisted_conversation_settings`. Both
     `OrgSettingsResponse.from_org` (line 193) and
     `OrgDefaultsSettingsResponse.from_org` (line 410) switch from
     `AgentSettings.model_validate(dict(org.agent_settings) if
     org.agent_settings else {})` to the loader. Loader handles
     the empty/None case internally.
   - `enterprise/storage/org_store.py:29, 57, 61` does the same
     for the bare `OrgStore.get_*` accessors. Consistent.

2. **SDK pin** to `054bbd4`. Touches every place that names a
   version:
   - `enterprise/poetry.lock`: `openhands-agent-server`,
     `openhands-sdk`, `openhands-tools` flip from `1.17.0` PyPI
     wheels to git refs at `054bbd4`. Resolved reference
     `054bbd4c40af4eb35af217a3ef85cf8e9f150136` recorded.
   - Container image tag in `containers/dev/compose.yml:16` and
     `docker-compose.yml:11`: `1.15.0-python` → `054bbd4-python`.
   - Pre-commit mypy `additional_dependencies` in
     `dev_config/python/.pre-commit-config.yaml:61-62` switched
     from pinned PyPI versions to the same git refs.

3. **Doc fix** in `AGENTS.md:41-43`: a paragraph explaining that
   when pinning to an unreleased SDK commit, the pre-commit mypy
   deps must be updated together — otherwise commit-time mypy
   installs the released SDK and fails on new APIs (specifically
   calls out `AgentSettings.from_persisted(...)`). Useful note
   for future SDK pins.

## Risks

- **All four pin sites must agree.** This PR appears to update all
  four (`poetry.lock` × 3 packages, dev compose, prod compose,
  pre-commit config). Easy to miss one in future bumps; the
  AGENTS.md note partially addresses this but a single helper
  script (`scripts/bump_sdk_pin.sh <sha>`) would be the durable
  fix.
- **`lmnr` upgrade slipped in** (`poetry.lock:4961-4974`,
  `0.7.46` → `0.7.48`). Reads as a transitive bump from
  `openhands-sdk`'s `lmnr >= 0.7.47` requirement (visible at
  line 6601). Fine, but worth calling out in PR body so reviewers
  don't wonder.
- **`develop = false`** on the three openhands packages
  (lock lines 6491, 6582, 6586) — that's poetry's standard for
  git-source deps, not a development-mode install. Just noting
  for reviewers who haven't seen it.
- **`tree_sitter_c_sharp` wheels added** for cp310-abi3
  (lock lines 14159-14166). Not obviously related to this change;
  likely a `poetry lock` regeneration side effect. Worth a
  sanity check that the lock-file regen was deliberate and
  no unintended packages were upgraded.
- The two helpers have leading underscores
  (`_load_persisted_agent_settings`, lines 18-21 of org_models.py).
  Importing private symbols across package boundaries is a
  brittle contract. If the SDK author meant these to be private,
  this PR is depending on internals; if they meant them public,
  they should be renamed without the underscore. Worth a
  conversation with the SDK side before merge.

## Suggestions

- Promote the two `_load_persisted_*` helpers to public names in
  the SDK and update this PR to import them without the underscore.
- Add a one-line note in PR body about the `lmnr` and tree-sitter
  bumps so reviewers don't have to dig.
- Consider extracting a `bump_sdk_pin` helper given how many sites
  need to stay in sync.

## What I learned

When you're pinning a downstream package to an unreleased commit,
the easy-to-miss dependency is the linter's own dependency tree.
Pre-commit installs its own mypy environment with its own
`additional_dependencies`, completely separate from your
`poetry install`. So `mypy` in CI passes against the new SDK
shape, but `pre-commit` fails locally because its mypy still has
the released SDK pinned. The AGENTS.md note here captures that
gotcha well — worth stealing into other repos that pin to git
refs.
