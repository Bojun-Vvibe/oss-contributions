# BerriAI/litellm#27147 — refactor: regenerate model_prices backup at build time

- Head SHA: `a2776fe7cadfe18090199a7e239d9bd1284557a5`
- Author: Chesars
- Verdict: **merge-after-nits**

## Summary

Stops shipping `litellm/model_prices_and_context_window_backup.json`
in the source tree and instead generates it at wheel-build /
Docker-build time from the canonical
`model_prices_and_context_window.json` at repo root. Replays a design
that was reverted in PR #16590, against the current release pipeline.
Deletes the ~40k-line backup JSON from VCS, deletes the
`ci_cd/check_files_match.py` drift-checker (no longer needed since
the backup is derived), and updates `get_model_cost_map.py` to load
from repo-root in source checkouts and from the bundled package
artifact in installed wheels.

## Specific references

- `Dockerfile:50-51`, `docker/Dockerfile.alpine`,
  `docker/Dockerfile.database`, `docker/Dockerfile.dev`,
  `docker/Dockerfile.non_root` all add the same incantation right
  after `COPY . .`:
  ```
  # Generate the model-cost backup artefact inside the package before the wheel build (mirrors publish_to_pypi.yml).
  RUN cp model_prices_and_context_window.json litellm/model_prices_and_context_window_backup.json
  ```
- `litellm/litellm_core_utils/get_model_cost_map.py:38-58` —
  `load_local_model_cost_map` now resolves repo root by walking up
  two parents from the source file and gating on `pyproject.toml`
  presence as a witness to avoid misidentifying an unrelated parent
  dir; falls back to `importlib.resources.files("litellm")` for
  installed wheels.
- `litellm/litellm_core_utils/get_model_cost_map.py` — new
  `FileNotFoundError` with an explicit operator-facing message that
  names the missing file, the workflow that materializes it
  (`.github/workflows/publish_to_pypi.yml`), and the manual
  `cp` command to recover.
- `ci_cd/check_files_match.py` — deleted in full; no longer needed.
- `tests/local_testing/test_get_model_file.py` — replaces the
  `importlib.resources.open_text` direct-load test with a call
  through the public `GetModelCostMap.load_local_model_cost_map()`
  classmethod, which exercises the new dual-source code path.

## Reasoning

This is the right direction. Shipping two byte-identical 40k-line
JSONs in the same repo is a perpetual source of merge conflicts and
silent drift; the deleted `check_files_match.py` was a band-aid that
auto-fixed drift and hid the underlying redundancy. The new
"generate at build time, witness via `pyproject.toml`" approach is
the conventional shape for this problem.

Concerns / nits to address before merge:

1. **`pyproject.toml` witness is fragile in monorepo / vendored
   contexts.** If a downstream consumer vendors `litellm/` into
   their own package and happens to have a `pyproject.toml` two
   directories above, the source-checkout branch fires and they get
   a `FileNotFoundError` for `model_prices_and_context_window.json`
   that they were never expected to provide. Tighter witness:
   require `pyproject.toml` *and* assert the `name = "litellm"`
   line, or look for a marker file like
   `model_prices_and_context_window.json` itself plus a
   `litellm/__init__.py`. Today's check is one regex away from
   safer.

2. **Local `pip install -e .` flow.** Mention in CONTRIBUTING (or
   add a `make` target) that an editable install needs the same
   `cp` step before first use, otherwise the installed-wheel
   branch fires (no backup file in the package), and the
   source-checkout branch also doesn't fire (because the editable
   install layout doesn't necessarily place
   `Path(__file__).parents[2]` at the repo root). The new
   `FileNotFoundError` message is good but it's only seen once
   the user is already broken.

3. **Five Dockerfiles with duplicated `RUN cp ...` lines.** Pull
   that into the `build_admin_ui.sh` neighbor or a tiny
   `docker/prepare_build_artifacts.sh` so the next time the
   recipe changes (e.g., adding a second derived artifact), you
   don't have to update five files.

4. **Test coverage gap.** The new
   `test_load_local_model_cost_map` only exercises the
   "happy path." Add a unit test that mocks
   `Path.is_file()` to simulate the installed-wheel branch and
   another that simulates the missing-witness case, asserting the
   `FileNotFoundError` message includes the recovery `cp` command.

5. **One-time release-PR coordination.** Because the backup file
   is being deleted from VCS in this PR, any in-flight PR that
   touches it will get a merge conflict that's likely
   misinterpreted as "I need to add the backup back." A note in
   the PR description (or a sticky issue) listing the specific
   in-flight PRs to rebase would prevent confusion.

The structural change is correct; the nits are about hardening
the source-vs-installed branching and reducing Dockerfile drift.
