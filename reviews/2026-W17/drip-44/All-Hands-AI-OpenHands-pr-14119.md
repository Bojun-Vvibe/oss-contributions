# All-Hands-AI/OpenHands PR #14119 — Removed V0 third party runtimes

- **URL:** https://github.com/All-Hands-AI/OpenHands/pull/14119
- **Author:** @tofarr
- **State:** MERGED 2026-04-24 (~21m after open)
- **Base:** `main`
- **Head SHA:** `14c716f`
- **Size:** +13 / -2,301

## Summary of change

Removes the `third_party/` runtime tree (Daytona, e2b, Modal, Runloop)
that supported the V0 application server. Aligns with the broader
V0 → V1 migration tracked in #14117 (V0 runtime removal) and #14122
(sub-agent delegation in app-server / V1).

Specific deletions:

- `containers/app/entrypoint.sh` L23–35: the `INSTALL_THIRD_PARTY_RUNTIMES`
  branch, which `pip install`'d `openhands-ai[third_party_runtimes]`
- `enterprise/poetry.lock` `[package.extras]` entry:
  `third-party-runtimes = ["daytona (==0.24.2)", "e2b-code-interpreter
  (>=2)", "modal (>=0.66.26,<1.2)", "runloop-api-client (==0.50)"]`
- `openhands/runtime/__init__.py`: rename `_DEFAULT_RUNTIME_CLASSES`
  to `_ALL_RUNTIME_CLASSES`, drop the dynamic `importlib`-based
  third-party-runtime registration block (~50 lines)
- `dev_config/python/mypy.ini`: drop `third_party/` from the exclude
  regex (matched in sibling PR #14120's lint configs)
- `.agents/skills/update-sdk/references/docker-image-locations.md`:
  drops the `third_party/runtime/impl/daytona/README.md` reference

## Findings against the diff

- **`openhands/runtime/__init__.py` L7–28**: the rename
  `_DEFAULT_RUNTIME_CLASSES` → `_ALL_RUNTIME_CLASSES` is doing real
  semantic work — previously the dict held only the always-available
  runtimes and the third-party importer extended it at startup; now
  it's literally the full set. Future readers will not be surprised
  by an `_ALL_` name that has only six entries.
- **`containers/app/entrypoint.sh` L23–35 deletion**: removes the
  optional `pip install 'openhands-ai[third_party_runtimes]'` step
  guarded by `INSTALL_THIRD_PARTY_RUNTIMES=true`. Worth confirming
  no published Helm chart / docker-compose preset still sets this
  env var — if any do, container startup won't fail (the env var is
  now ignored), but operators may have a quiet expectation that those
  runtimes are still available.
- **`enterprise/poetry.lock` [package.extras]**: dropping the
  `third-party-runtimes` extra is the correct downstream of removing
  `third_party/` from source. If a tagged release of `openhands-ai`
  on PyPI still advertises this extra, `pip install
  'openhands-ai[third_party_runtimes]'` will fail loudly with
  "extra not provided" once a new release ships — that's the right
  failure mode (loud, immediate, easy to understand).
- **`get_impl()` consumers**: `openhands/utils/import_utils.py` is
  not in this diff, but it's the lookup layer that reads
  `_DEFAULT_RUNTIME_CLASSES` (now `_ALL_RUNTIME_CLASSES`). Sibling
  PR #14117 needs to be checked for the rename match; if that PR
  landed first with the old name, there's a brief window of
  inconsistency. The 21-minute merge window suggests the stack was
  squashed cleanly.
- **No deprecation period.** This is "delete V0, V1 is the path
  forward." Acceptable because (a) `runtime/__init__.py` already
  carried the `# Tag: Legacy-V0` comment from a prior round, and
  (b) the V0 → V1 migration has been tracked publicly. Users who
  pinned to `daytona`/`e2b`/`modal`/`runloop` runtimes will need to
  pin to a pre-#14119 release until they migrate.

## Verdict

**merge-after-nits** (already merged, so the nits are post-merge
follow-ups)

The bulk of the change is correct and consistent with the V0-removal
program. Post-merge follow-ups worth opening:

1. Add a NEWS / migration note pointing third-party-runtime users to
   the last release that supported them, plus the V1 equivalents.
2. If `containers/app/entrypoint.sh`'s `INSTALL_THIRD_PARTY_RUNTIMES`
   was documented anywhere user-facing (Helm values, docker-compose
   examples), scrub those references.
3. Consider whether `_ALL_RUNTIME_CLASSES` should become a frozen
   `MappingProxyType` now that the import-side extension hook is
   gone — protects against accidental runtime mutation, which used
   to be a feature and is now an antipattern.

## What I learned

Deletions in the 2k-line range are usually safer than additions in
the 200-line range, because the cost surface is shrinking instead of
growing. The trick is to make sure all the *naming* survives the
deletion: the `_DEFAULT_` → `_ALL_` rename in this diff is doing more
work than the line count suggests, because every grep that hits the
old name now needs to follow the rename, and the new name now means
something stronger ("this is the full set, not a default") that
constrains future contributors. A delete-PR that only deletes is
usually fine; a delete-PR that also renames is a delete-PR plus a
small refactor, and the rename should be reviewed on its own merits.
