---
pr-url: https://github.com/google-gemini/gemini-cli/pull/26285
sha: f497240f7ea6
verdict: merge-after-nits
---

# fix(cli): use resolved sandbox state for auto-update check

Source-of-truth refactor: `handleAutoUpdate` at `packages/cli/src/utils/handleAutoUpdate.ts:68-82` grows a new `isSandboxEnabled: boolean` parameter (4th positional, before the existing `spawnFn` injection slot), and the load-bearing line at `:81` flips from `if (settings.merged.tools.sandbox || process.env['GEMINI_SANDBOX'])` to `if (isSandboxEnabled)`. The single production caller at `packages/cli/src/interactiveCli.tsx:182` passes `config.getSandboxEnabled()` rather than the inlined `settings || env-var` expression. The 13 test sites at `handleAutoUpdate.test.ts` are mechanically updated to insert `false` as the new positional argument before the existing `mockSpawn`, plus one new test at `:404-411` asserting `handleAutoUpdate(mockUpdateInfo, mockSettings, '/root', true, mockSpawn)` emits `update-info` with the appended `\nAutomatic update is not available in sandbox mode.` text and does not invoke `spawn`.

The fix is the right shape — the bug it addresses is that `settings.merged.tools.sandbox || process.env['GEMINI_SANDBOX']` is a *re-derivation* of the sandbox-enabled state at the call site, which can disagree with whatever `config.getSandboxEnabled()` resolved at startup time (config initialization runs through `--sandbox` CLI flag, settings precedence, profile-mode overrides, sandbox-image fallback detection — all of which can produce a sandbox-enabled state that the inline OR-expression doesn't capture). Centralizing the resolution at config-init and threading the resolved bool down through the call chain is the standard fix for this "two sources of truth that have drifted" bug class.

Three nits, none big enough to gate:

(1) Positional 4th-argument insertion in front of `spawnFn` is the wrong direction for ergonomics — every caller now has to remember "is the bool or the spawn fn the 4th argument?" and the test diff has to mechanically thread `false` through 13 call sites. An options-bag (`{ isSandboxEnabled, spawn }`) would have made the test-site churn one diff-hunk per file rather than 13, and would have been future-proof against the next required parameter.

(2) The PR removes the `process.env['GEMINI_SANDBOX']` env-var check from this code path entirely — that's correct *if* `config.getSandboxEnabled()` already incorporates the env-var, but the diff doesn't show or assert that. A grep for `GEMINI_SANDBOX` in the config-resolution path, or a 5-line test at the config layer pinning "env-var sets sandbox-enabled," would close the loop. Without it, a future "let me drop the env-var honoring" config-refactor will silently regress this fix.

(3) The new sandbox-true test at `:404-411` asserts the `update-info` emission and the absence of `spawn`, but doesn't pin the *order* (info-emit must precede the spawn-suppress decision in the function so the user sees the message before the silent no-op). A `vi.mock` call-order assertion would make the contract explicit.

## what I learned
"Source of truth at the call site re-derives state that's already been resolved upstream" is the most common subclass of state-drift bug in CLI codebases, and the fix is almost always to add a parameter rather than to reach back up to the resolution function — but the parameter shape matters, and 4th-positional-bool ahead of an existing function-injection slot is a regret pattern that will bite the next reviewer. Options-bag from the start avoids both the test-churn cost and the call-site readability cost.
