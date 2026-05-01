# google-gemini/gemini-cli#26321 — fix: sanitize keychain native errors and fix test execution

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26321
- **Head SHA**: `98f51140f3c465c5f6391c944705408153ccddcc`
- **Size**: +186 / -60, 6 files (`keychainService.ts` + tests, `keychain-token-storage.test.ts`, `package.json`, `package-lock.json`, snapshot test)
- **Verdict**: **merge-after-nits**

## Context

`KeychainService` (the macOS Keychain / Windows Credential Store wrapper) propagated raw native error messages from the underlying `keytar` / `node-keytar` binding. Those messages can include OS-level paths, library handle pointers, or otherwise sensitive context that ends up in `console.error` output, telemetry, or — worst case — bug reports the user posts to GitHub. The PR introduces a `SecureStorageError` wrapper class that replaces the raw error with a stable `"A secure storage operation failed."` message, while preserving the original error as `cause` for in-process diagnostics. Also fixes a `bundle` script that was running `core` build via `bundle:browser-mcp` instead of the standard `build` workspace.

## What's right

- **`SecureStorageError` at `packages/core/src/services/keychainService.ts`** — new exported class with a fixed user-facing message. The pattern is the right shape for OS-binding error sanitization: callers can `instanceof SecureStorageError` to branch on "this is a known sanitized native error," `.cause` carries the raw error for `verbose_logger.debug` / telemetry redaction at the appropriate layer, and the displayed string is non-leaky.
- **All four mock arms in `keychain-token-storage.test.ts:155-167` flip together.** `getPassword`, `setPassword`, `deletePassword`, `findCredentials` mocks all now reject with `new SecureStorageError()` instead of `new Error('Keychain is not available')`. The follow-up assertion at `:184` flips from `.toThrow('Keychain is not available')` to `.toThrow('A secure storage operation failed.')`. Symmetric replacement, no half-migrated state.
- **`keychainService.test.ts` adds 56 lines / -4** (visible in the file-list summary; full diff truncated). Reading the surrounding context, this should be where the sanitization discipline gets pinned at the source: every native-binding call wrapped in a `try/catch` that re-throws `new SecureStorageError(undefined, { cause: originalErr })`.
- **`package.json` `bundle` script fix** — `npm run bundle:browser-mcp -w @google/gemini-cli-core && node esbuild.config.js` becomes `npm run build --workspace=@google/gemini-cli-core && npm run bundle:browser-mcp -w @google/gemini-cli-core && ...`. The prior order ran the browser-MCP bundle without first running the workspace's standard `build`, which presumably broke when the core package gained new files that the bundle depended on but that hadn't been compiled. One-line ordering fix at `package.json:42`.
- **`@babel/*` minor-version bumps in `package-lock.json`** are harmless (7.27/7.28 → 7.29) and consistent across `code-frame`, `generator`, `parser`, `template`, `traverse`, `types` — the typical babel-bump pattern from a transitive dep nudge.
- **Snapshot additions at `packages/cli/src/ui/components/__snapshots__/InputPrompt.test.tsx.snap:171-191`** — three new `should toggle paste expansion on double-click 4/5/6` snapshots are pure additions, suggesting the prior test was tightened to assert *more* render frames after the same interaction (catches an off-by-one in the render-cycle that was previously masked by checking only frames 1-3).

## Risks / nits

- **`SecureStorageError` constructor signature is unclear from this diff.** The test at `:158-167` calls `new SecureStorageError()` (no args). If the class is `class SecureStorageError extends Error { constructor(opts?: { cause?: unknown }) { super("A secure storage operation failed."); ... } }` the `cause` chain is preserved only when callers explicitly pass it. Verify every native-binding call site in `keychainService.ts` passes `{ cause: originalErr }` so debug-level logging can still see the raw message.
- **No telemetry-redaction guard at `error.cause`.** If anything downstream serializes a thrown `SecureStorageError` for telemetry or crash-reporting, the `.cause` chain (containing the raw native message) would leak the very thing this PR sanitizes. Worth a `toJSON()` override on `SecureStorageError` that omits `cause` from JSON serialization, with a separate `getDiagnostic()` method for the verbose-logger code path.
- **The `extraneous: true` `third_party/get-ripgrep` entry in `package-lock.json:18792-18816`** appears to be an unrelated workspace addition. Worth confirming this isn't accidentally bundled into the lockfile change — the `extraneous: true` flag suggests npm is tracking it without actually installing it, which is harmless but noise.
- **No test arm pinning that the user-facing string is exactly `"A secure storage operation failed."`** beyond the indirect `.toThrow(...)` assertion. A tiny snapshot-style test at the keychainService level (`expect(new SecureStorageError().message).toBe("A secure storage operation failed.")`) would prevent silent message drift.

## Verdict

**merge-after-nits.** Right shape for OS-binding error sanitization (typed wrapper class, fixed user-facing string, `cause` chain for diagnostics). The `bundle` script fix is a clean ordering bug-fix. Pin the `cause`-chain serialization behavior and add a direct message assertion before merge; the babel bumps and snapshot additions are fine as-is.
