# browser-use/browser-use PR #4737 — Fix Windows CLI session probe timeouts

- **URL:** https://github.com/browser-use/browser-use/pull/4737
- **Head SHA:** `ab0db0bcbd2f7cf4de40ae1b50c533c4faff5359`
- **Files touched:** 5 (`skill_cli/main.py`, `skill_cli/utils.py`, two tests, `.pre-commit-config.yaml`)
- **Verdict:** `merge-after-nits`

## Summary

Fixes a class of Windows-only CLI hangs where the daemon probe blindly
trusted any reachable TCP endpoint and could spin forever on
unresponsive sockets. Adds short timeouts for `ping`, session listing,
and shutdown control calls; namespaces Windows TCP daemon ports by
`BROWSER_USE_HOME + session` so concurrent homes can't collide.

## Specific references

- `browser_use/skill_cli/main.py:171-177` — `_get_socket_path` builds the
  Windows port from `casefold()`-ed home dir + session name and feeds it
  through `zlib.adler32`. This is the right level of namespacing:
  per-user-home, deterministic, no extra state file.
- `browser_use/skill_cli/main.py:200-202` — new module-level constants
  `_DAEMON_PROBE_TIMEOUT = 1.0` and `_DAEMON_CONTROL_TIMEOUT = 2.0`. Good
  to have these named instead of inline.
- `browser_use/skill_cli/utils.py:+3` — kept aligned with `main._get_socket_path`,
  which prevents the foot-gun where one of the two diverges.
- `tests/ci/test_cli_lifecycle.py:+151` — `test_probe_session_requires_successful_ping`
  is the key regression: a TCP endpoint that accepts the connection but
  never replies must be treated as dead, not reusable.
- `tests/ci/test_cli_sessions.py:+58` — `test_windows_socket_path_is_namespaced_by_home`
  asserts two homes hash to different ports.
- `.pre-commit-config.yaml:1-3` — drops `default_language_version: python:
  python3.11`. Worth a sanity check that CI runners still pin a sensible
  Python; quietly removing version pins in pre-commit configs has burned
  this repo before.

## Reasoning / nits

- The home-key uses `casefold()` of `Path.resolve()`. On Windows this is
  fine; on POSIX `Path.resolve()` already canonicalises and casefold is a
  no-op. Behaviour is consistent.
- `adler32` is fine for collision-avoidance across homes but is *not*
  cryptographic — confirm in review that the port is also gated by the
  per-session auth token (it is, via `_read_auth_token`). Looks safe.
- Removing the python3.11 pin in pre-commit (line 1-3 of the config diff)
  is the only line that doesn't have an obvious justification in the PR
  body. Ask author whether this was intentional or accidental drift.

Merge after the pre-commit pin question is answered.
