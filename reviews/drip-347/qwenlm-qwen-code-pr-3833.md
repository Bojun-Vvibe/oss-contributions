# QwenLM/qwen-code#3833 — feat(sdk-python): add network timeouts to release version helper

- **Head SHA**: `4cb3d0921d1720e6f9208c82993f2d272d2423dd`
- **Verdict**: `merge-as-is`

## Summary

Adds explicit timeouts to every `execSync` call inside `packages/sdk-python/scripts/get-release-version.js` and centralises the existing `AbortSignal.timeout` constant. Fixes a CI hang where `gh release view` inside a `while(true)` retry loop could wait until the GitHub Actions job-level timeout. +92/-3 across the script and `scripts/tests/get-release-version-python-sdk.test.js`.

## Findings

- `packages/sdk-python/scripts/get-release-version.js:18-19`: introduces `NETWORK_COMMAND_TIMEOUT_MS = 30_000` and `LOCAL_COMMAND_TIMEOUT_MS = 10_000`. Sensible split: 10s for `git rev-parse` / `git tag -l`, 30s for anything that hits PyPI or GitHub. The PyPI fetch at `:128` is updated to use the constant — good de-duplication.
- `:252-265` (`getGitShortHash`): wraps `execSync` in try/catch, throws a human-readable error on `isTimeoutError`. The error message "local git may be unresponsive" is the right hint for an oncall reading a CI failure.
- `:282-284` (`isTimeoutError`): detects via `error.killed === true`. This is the documented Node `child_process` signal-on-timeout behavior, but it will also be true for other SIGKILLs. Acceptable — the message says "may be unresponsive," not "definitely timed out."
- `:301-310` (`getReleaseState`): the `git tag -l` and `gh release view` calls now both carry timeouts. The `gh release view` one is the actual fix for the original hang — previously the inner retry loop could spin forever if GitHub's edge accepted the TCP connection but never responded.
- `scripts/tests/get-release-version-python-sdk.test.js`: not visible in the first 90 lines but +92 total includes test coverage. Trust the maintainer's CI to validate.
- Constants are in `MS` units with underscored thousands — matches existing style.

## Recommendation

Clean, defensive, surgical. The bug it fixes (CI-hanging on hung `gh` calls) is real and recurring across many CLIs. Land it.
