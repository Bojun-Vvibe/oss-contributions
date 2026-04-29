# google-gemini/gemini-cli #26162 — Test env-leak hardening (`ANTIGRAVITY_CLI_ALIAS`, `GEMINI_CLI_HOME`)

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26162
- **Size:** +5 / 0 across 3 test files

## Summary
Adds `vi.stubEnv(...)` calls in `beforeEach` of three test files to neutralize ambient `ANTIGRAVITY_CLI_ALIAS` and pin `GEMINI_CLI_HOME` to the per-test temp directory. Pure test-isolation hardening — no production code change.

## Specific observations

1. **Three files touched, all in `beforeEach`:**
   - `packages/cli/src/config/extension-manager-agents.test.ts:45,49` — adds `vi.stubEnv('ANTIGRAVITY_CLI_ALIAS', '')` and `vi.stubEnv('GEMINI_CLI_HOME', tempDir)` after `mockHomedir.mockReturnValue(tempDir)`.
   - `packages/cli/src/config/extension-manager-hydration.test.ts:51,55` — same pattern.
   - `packages/core/src/core/contentGenerator.test.ts:43` — `vi.stubEnv('ANTIGRAVITY_CLI_ALIAS', '')` only (no `GEMINI_CLI_HOME` because this test doesn't stub homedir).

2. **Why this matters:** these test suites read configuration from paths derived from `os.homedir()` (mocked) and from env-var-influenced search paths. If a developer or CI runner has `ANTIGRAVITY_CLI_ALIAS` set (which is plausible if they've installed any tool that uses that env var globally), the test reads alias config from outside the temp dir, producing flaky pass/fail depending on the developer's shell. Same for `GEMINI_CLI_HOME` — without the stub, the test honors a real `GEMINI_CLI_HOME` env, blowing past the homedir mock entirely.

3. **`vi.stubEnv(name, '')` (empty string) is the right approach** — vitest's `stubEnv` with `''` makes the env var defined-but-empty, which most config readers treat as "not set." Setting it to an explicit value would be wrong (it'd substitute one ambient interference for another). Vitest auto-restores stubbed env on `vi.unstubAllEnvs()` (or test teardown if `unstubEnvs: true` is configured), so no leak across files.

4. **Asymmetry between the two test files:** `extension-manager-agents.test.ts` and `extension-manager-hydration.test.ts` get both stubs; `contentGenerator.test.ts` gets only `ANTIGRAVITY_CLI_ALIAS`. This is correct given the test surface area, but a future hardening pass might want to set both consistently in a shared `beforeEach` helper.

5. **No `afterEach` cleanup** — fine, because vitest's `vi.stubEnv` is auto-cleaned (verified by the fact that `vi.unstubAllEnvs()` is the standard companion and is implicit in vitest's per-test isolation model with `unstubEnvs: true`, which is the default in recent vitest versions). If the project's `vitest.config.ts` has `unstubEnvs: false`, this PR introduces a leak — worth a quick check.

## Risks

- **`unstubEnvs: false` config gap:** if the project explicitly disables env-stub auto-cleanup, the `ANTIGRAVITY_CLI_ALIAS` and `GEMINI_CLI_HOME` stubs leak into subsequent test files and could destabilize unrelated suites. Verify `vitest.config.ts` defaults.
- Trivial otherwise. Pure test-isolation tightening, no production behavior change.

## Suggestions

- **(Recommended)** If the same env-stub pattern is repeated across more test files, factor it into a shared `setupTestEnv()` helper that both stubs and asserts the values are restored at teardown.
- **(Optional)** Add a top-of-file comment on each test pointing at this PR or the bug ID, so the next developer doesn't delete the stubs as "dead lines."
- **(Nit)** Consider stubbing `ANTIGRAVITY_CLI_ALIAS` once in a global vitest setup file rather than per-suite — env-leak hardening is a workspace-level concern.

## Verdict: `merge-as-is`

Pure test-isolation fix, mechanically correct, zero production impact. The +5/-0 footprint is exactly the right size for what it does.

## What I learned

Environment-variable-driven config is a recurring source of test flakiness, and the diff between "passes locally, fails in CI" vs "passes in CI, fails on a developer's box with `ANTIGRAVITY_CLI_ALIAS=foo` in their shell" is almost always one missing `vi.stubEnv` call. The right defense is at the test-framework-config level (vitest's `unstubEnvs: true` default plus a workspace-wide setup file that stubs every config-influencing env var to empty), not per-test patches like this one. This PR is the right local fix; the structural fix is a follow-up.
