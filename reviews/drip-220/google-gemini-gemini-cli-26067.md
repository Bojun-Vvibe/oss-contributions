---
pr-url: https://github.com/google-gemini/gemini-cli/pull/26067
sha: a15568e013d8
verdict: merge-after-nits
---

# fix(cli): correct alternate buffer warning logic for JetBrains

Two-site fix at `packages/cli/src/gemini.tsx`. Removes the `isAlternateBufferEnabled(config)` import (deleted line 88) and replaces the call site at `:646` with `config.getUseAlternateBuffer()` directly. The companion test addition at `packages/cli/src/utils/userStartupWarnings.test.ts:223-238` adds an `it('should correctly pass isAlternateBuffer option to getCompatibilityWarnings'...)` arm asserting both `true` and `false` round-trip through `getUserStartupWarnings({}, projectDir, { isAlternateBuffer: ... })`.

The bug shape: `isAlternateBufferEnabled` was a wrapper that read multiple inputs (terminal capability detection plus the config flag) and returned a *resolved* boolean. JetBrains' embedded terminal advertises capabilities that made the wrapper return `true` even when the user/config wanted alt-buffer disabled, so `shouldEnterAlternateScreen(true, screenReader)` triggered the alt-screen entry path and the JetBrains terminal renderer (which has known issues with the `\e[?1049h`/`\e[?1049l` sequences) garbled output. Reading `config.getUseAlternateBuffer()` directly bypasses the capability-detection layer and respects the user's explicit setting — which is the right precedence for an *opt-in* alt-buffer feature.

The fix is correct in the *narrow* sense (the user-visible JetBrains regression goes away) but the deletion of the wrapper without checking other call sites is risky:

1. **No `grep isAlternateBufferEnabled` evidence in the PR description.** If any other site still imports the wrapper they now read a stale predicate, and if no other site imports it the wrapper itself is dead code that should be deleted in the same PR (otherwise the next reader will assume it's still load-bearing somewhere).
2. **The new test asserts plumbing, not behaviour.** `expect(getCompatibilityWarnings).toHaveBeenCalledWith({ isAlternateBuffer: true })` pins the mock-call signature, which catches "we forgot to pass the flag" but not "JetBrains terminal still gets the warning suppressed when alt-buffer is on." A higher-fidelity test would mock the JetBrains-detection signal and assert the warning *appears* in the JetBrains+alt-buffer arm and *doesn't* appear in JetBrains+no-alt-buffer.
3. **The `gemini.tsx:646` call site has no comment** explaining why the direct config read is preferred over the wrapper. A future reader looking at the wrapper definition will reasonably assume it's the canonical entry point and re-introduce it here, undoing the fix. A two-line comment "Read the user's explicit setting; do not consult terminal capability detection because [JetBrains issue #...]" would prevent regression.

## what I learned
Capability-detection wrappers are tempting because they look like they encode "expert knowledge about which terminals support which features" — but in practice they accumulate a tail of false positives (terminals that lie about their capabilities) and false negatives (terminals that work fine but fail the heuristic). The correct precedence for opt-in features is *user setting wins, capability detection only gates the default*. Wrappers that erase that distinction silently override user intent, which is the worst possible UX: the user changes a setting and nothing happens, with no error to grep for.
