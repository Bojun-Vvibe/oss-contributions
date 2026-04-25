# PR #19462 — sdk/python: use standalone codex-app-server runtime

- Repo: openai/codex
- Head SHA: 6aa4aed2024f9dc3ea35e3eaba57ec7af0271541
- Files touched: 14 (+521 / -206)

## Summary

Python SDK follow-up to #19447 (drip-25, codex-app-server release
artifact split). Renames the platform runtime wheel from
`openai-codex-cli-bin` → `openai-codex-app-server-bin`, leaves a
compatibility shim at `codex_cli_bin` that re-exports
`bundled_app_server_path`, teaches `_runtime_setup.py` to prefer
`codex-app-server-*` GitHub release assets and fall back to legacy
`codex-*` assets, switches the launcher to resolve the standalone
binary, adds `AppServerConfig.app_server_bin` (preferred) while
keeping `codex_bin` as an alias, and **raises** when callers try to
combine `config_overrides` with the standalone binary path.

## Specific findings

- `sdk/python-runtime/src/codex_app_server_bin/__init__.py:1-31`
  introduces `bundled_app_server_path()` with a candidate
  ordering of `("codex-app-server", "codex")` (Unix) /
  `("codex-app-server.exe", "codex.exe")` (Windows). Falling back
  to `codex` is what makes the older runtime wheel usable when
  the SDK gets pinned to a fresh runtime version before the
  legacy `codex` binary is purged — defensible but the candidate
  list should be narrowed to `("codex-app-server",)` once the
  legacy wheel is no longer staged, otherwise a stale wheel will
  satisfy the lookup with the wrong binary forever.
- `sdk/python-runtime/src/codex_cli_bin/__init__.py:1-13` reduces
  to a thin re-export. `bundled_codex_path()` now returns
  `bundled_app_server_path()` — that's the right shape for a
  compatibility shim but reviewer should confirm no internal
  caller still relies on `codex_cli_bin.PACKAGE_NAME` matching
  the *old* wheel name (`openai-codex-cli-bin`); after this PR
  it returns the new package name, which can confuse consumers
  doing string-based version reporting.
- `sdk/python-runtime/hatch_build.py:18-26` keeps the legacy
  `CODEX_CLI_BIN_PLATFORM_TAG` env as a fallback to the new
  `CODEX_APP_SERVER_BIN_PLATFORM_TAG`. That dual lookup is
  worth a deprecation warning when only the legacy var is
  present so CI surfaces start switching.
- `sdk/python/src/codex_app_server/client.py` (additions:108,
  deletions:39) adds `app_server_bin` and the explicit raise on
  `config_overrides + standalone codex-app-server`. A clear
  exception is correct here — the standalone binary doesn't
  accept the legacy CLI startup flags, so silent ignore would
  surprise callers that depended on overrides taking effect.
- `sdk/python/scripts/update_sdk_artifacts.py` (additions:22,
  deletions:7) updates the `stage-runtime` subcommand path to
  point at `/tmp/codex-python-release/openai-codex-app-server-bin`
  and the source binary at `/path/to/codex-app-server`. CI flow
  needs the equivalent rename in any wrapper scripts that hard-
  code the old wheel name.
- `sdk/python/tests/test_artifact_workflow_and_binaries.py`
  (additions:159, deletions:32) covers new runtime naming, the
  standalone launch path, and the legacy fallback path. The PR
  body excludes `test_generated_files_are_up_to_date` because
  it's drifting on `main`; that's a fair pragmatic call but the
  drift should be tracked separately.

## Risks / nits

- The `config_overrides + codex-app-server` raise is a hard
  break for any caller that was passing both. Worth a release-
  note bullet calling out the migration path
  (`AppServerConfig.codex_bin=...` to keep legacy semantics).
- The fallback chain in `_runtime_setup.py` can satisfy a fresh
  SDK pin against a stale runtime wheel that contains only the
  old `codex` binary. That's deliberate per the PR body but
  needs an explicit deprecation window — otherwise the legacy
  asset path will accumulate forever.

## Verdict

**merge-after-nits** — Clean migration shape with a real
compatibility story. Address the legacy-env-var deprecation
warning and the `codex_cli_bin.PACKAGE_NAME` rename surprise
before tagging an SDK release.

## What I learned

When you rename a wheel that downstream callers reach into via
`from <package> import PACKAGE_NAME`, the value of `PACKAGE_NAME`
is itself part of the public API. Re-exporting it from a thin
shim is the right shape, but you have to decide whether the
shim's `PACKAGE_NAME` keeps the old value (preserves observable
behavior, costs introspection accuracy) or adopts the new value
(matches reality, but breaks any caller doing string-based
version assertions). This PR picks the latter — defensible, but
the migration note has to be loud.
