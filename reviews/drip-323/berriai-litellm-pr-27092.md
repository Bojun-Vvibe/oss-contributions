# BerriAI/litellm #27092 — fix(secret_managers): contain oidc/env_path file reads to allowlisted dirs (CWE-22)

- **PR**: https://github.com/BerriAI/litellm/pull/27092
- **Head SHA**: `5e8a251bf5b72456bf2a1238a0b5bdc6f1ba9b13`
- **Author**: sebastiondev
- **Size**: +25 / −7, 2 files

## Files changed

- `litellm/secret_managers/main.py` (+3 / −1)
- `tests/litellm_utils_tests/test_secret_manager.py` (+22 / −6)

## Summary of change

Closes a path-traversal / arbitrary-file-read in `get_secret()`'s
`oidc/env_path/` branch. The sibling `oidc/file/` branch already routes
through `_resolve_oidc_file_path()` (which `realpath`s and verifies
`commonpath` against `LITELLM_OIDC_ALLOWED_CREDENTIAL_DIRS`); the
`env_path` branch did not. The fix is a one-line containment:
`safe_path = _resolve_oidc_file_path(token_file_path)` before the `open()`.

## Specific-line review

- `litellm/secret_managers/main.py:260-263` (post-diff): the new
  `safe_path = _resolve_oidc_file_path(token_file_path)` is inserted
  immediately after the `os.getenv` lookup and before `open()`. This is
  exactly the right spot — no TOCTOU window between resolve and open
  beyond what already exists in the `oidc/file/` branch, so we are not
  introducing a new race.
- The vulnerability description in the PR body is accurate: `get_secret()`
  is reachable from request-supplied `azure_ad_token` /
  `aws_web_identity_token` (popped from kwargs in `litellm/main.py`), and
  the `env_path` branch was the only OIDC sub-branch missing the
  containment. The CWE-22 framing is correct.
- `tests/.../test_secret_manager.py` — `test_oidc_env_path` now sets
  `LITELLM_OIDC_ALLOWED_CREDENTIAL_DIRS` and writes the token inside that
  dir (positive path). The new
  `test_oidc_env_path_rejects_paths_outside_allowed_dirs` writes outside
  the allowlist and asserts the `ValueError` containing `"outside the
  allowed credential directories"`. Both happy-path and rejection-path are
  covered. `pytest.raises(..., match=...)` is a string-stable assertion;
  if the upstream message ever changes the test will fail loudly, which is
  fine.

## Risk / compat

- Behavior change: any deployment that today relies on `oidc/env_path/`
  pointing at a file outside `/var/run/secrets` or `/run/secrets` (or the
  configured allowlist) will start failing with `ValueError`. This is a
  feature, not a bug, but should be called out in the changelog so
  operators know to extend `LITELLM_OIDC_ALLOWED_CREDENTIAL_DIRS` if their
  layout differs.
- The PR author already flagged a related sink at
  `litellm/proxy/auth/rds_iam_token.py:80` that opens an
  `aws_web_identity_token` path directly. That value is server-side
  config, not request-derived, so leaving it for a follow-up PR is the
  right call.

## Verdict

**merge-after-nits** — the fix itself is correct and well-tested. Nit:
add a one-liner to the changelog / release notes describing the
behavior change (rejected paths now raise `ValueError`) and the
`LITELLM_OIDC_ALLOWED_CREDENTIAL_DIRS` knob, so operators are not
surprised on upgrade.
