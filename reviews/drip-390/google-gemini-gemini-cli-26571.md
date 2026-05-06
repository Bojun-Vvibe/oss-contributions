# Review: google-gemini/gemini-cli#26571 — fix(core): prevent silent hang during OAuth auth on headless Linux

- Head SHA: `11a59c780291ecc7e521a030ffec73c31a6f7dcf`
- Files: 2 (+26 / -15)
- Verdict: **merge-after-nits**

## Summary

Two compounding fixes for the silent hang where `gemini` deadlocks early in startup on headless Linux (WSL/SSH/Docker/CI) with `oauth-personal` selected and no cached credentials: (1) skip the unused `loadApiKey()` call for OAuth auth types, (2) bound the keychain functional probe with a 2s timeout that falls back to `FileKeychain` on timeout.

## Specific evidence

- **Move early-return above the `loadApiKey()` await** at `packages/core/src/core/contentGenerator.ts:140-160`:
  ```ts
  // BEFORE: loadApiKey() ran for every auth type, even LOGIN_WITH_GOOGLE / COMPUTE_ADC
  // AFTER:
  if (
    authType === AuthType.LOGIN_WITH_GOOGLE ||
    authType === AuthType.COMPUTE_ADC
  ) {
    return contentGeneratorConfig;
  }
  const geminiApiKey = apiKey || process.env['GEMINI_API_KEY'] || (await loadApiKey()) || undefined;
  ```
  Correct: the early-return at `:148` was unreachable-after-`await` because the `await loadApiKey()` happened first. `geminiApiKey` is unused on the OAuth/ADC paths (the function returned at `:148` before reading it), so the work was always wasted. Now the OAuth path never touches the keychain layer at all.
- **Comment at `:142-143`** explicitly names the failure mode (Linux without Secret Service: WSL/SSH/Docker/CI) so future refactors don't re-flatten the early-return back below the load.
- **Bounded probe** at `packages/core/src/services/keychainService.ts:139-153`:
  ```ts
  const functional = await Promise.race([
    this.isKeychainFunctional(keychainModule),
    new Promise<false>((resolve) =>
      setTimeout(() => resolve(false), 2000).unref(),
    ),
  ]);
  if (functional) {
    return keychainModule;
  }
  ```
  `Promise.race` with a 2s deadline + `setTimeout(...).unref()` so the timer doesn't keep the event loop alive if the probe wins. Good defense.
- **Fallthrough is the same `return null` path** at `:154-155` that already triggers the `FileKeychain` fallback in the caller, so timeout → file-backed keychain is the existing well-tested code path. No new branch.
- **Symmetry check vs macOS**: the existing `isMacOSKeychainAvailable` (lines 193-220 per PR body) does an explicit `security default-keychain` pre-check to avoid blocking popups. Linux had no equivalent. The 2s race is the cross-platform analog. Correct asymmetry: macOS uses an explicit pre-check, Linux uses a bounded probe — both arrive at the same "fall back to FileKeychain" end-state.
- **Repro is reproducible from the PR body**: `gemini -p hi` with `selectedType: "oauth-personal"` and no `~/.gemini/oauth_creds.json` on a `SSH_CONNECTION`-set / `DISPLAY`-unset env hits `timeout 15` with 0 bytes of output. The `GEMINI_FORCE_FILE_STORAGE=true` workaround confirms the keytar probe is the deadlock site.

## Nits

1. **No new test added.** The behavior is hard to unit-test (it's a deadlock that depends on D-Bus/libsecret state), but a fake `keychainModule` whose `setPassword` returns a never-resolving Promise would let a unit test pin the 2s race outcome (`functional === false`) deterministically. Recommended.
2. **2s timeout is hardcoded.** Worth making it configurable via env (e.g. `GEMINI_KEYCHAIN_PROBE_TIMEOUT_MS`) so heavily-loaded CI runners with a slow but functional Secret Service don't get pushed into FileKeychain on a transient slow probe. Non-blocking.
3. **`debugLogger.debug('Keychain functional verification failed or timed out')`** at `:152` — the timeout case probably warrants `info` not `debug`, since a user who *expects* keychain-backed storage and silently got file storage will want the breadcrumb without enabling debug logging. Style call.
4. **The early-return reorder in `createContentGeneratorConfig`** changes the semantic of `apiKey` parameter precedence for the OAuth/ADC branches — but since those branches don't read `geminiApiKey`, the user-visible behavior is unchanged. Worth a one-line test pinning that calling `createContentGeneratorConfig({ authType: LOGIN_WITH_GOOGLE, apiKey: 'will-be-ignored' })` still returns a config with `apiKey === undefined`.
5. The PR body's "AI generated summary below" disclaimer is helpful provenance — no action needed.

Two real bugs, two minimal fixes, both grounded in a reproducible deadlock with a known workaround. Merge after a unit test on the bounded-probe path.
