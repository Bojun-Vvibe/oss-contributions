# google-gemini/gemini-cli PR #26322 — Sanitize keychain errors, restore test execution, fix install/build

- Head SHA: `999ba43c0dec23e34f57bd01f61ea943556cfe97`
- URL: https://github.com/google-gemini/gemini-cli/pull/26322
- Size: +140 / -30, 3 files
  (`packages/core/src/services/keychainService.ts`,
  `packages/core/src/services/keychainService.test.ts`,
  `packages/core/src/utils/errors.ts`)
- Verdict: **merge-after-nits**

## What changes

Two coupled improvements to the keychain abstraction:

1. **PII-scrubbing of keychain error messages.** Every public
   keychain method (`getPassword`, `setPassword`, `deletePassword`,
   `findCredentials`) is now wrapped in `runSecureStorageOperation`
   (keychainService.ts:264-275) which catches *any* underlying
   exception and rethrows a `SecureStorageError(SECURE_STORAGE_GENERIC_MESSAGE)`
   — losing the original message which on macOS often contains paths
   like `/Users/<name>/Library/Keychains/login.keychain-db` or
   account names from the keytar layer.
2. **Quieter init logging.** The catch arm in
   `getKeychain` (keychainService.ts:223-225) used to log
   `"Keychain initialization encountered an error:"` followed by the
   raw `error.message`. Now it logs only `"Keychain initialization
   failed"` (no payload). Same for the schema-validation arm
   (line 237).

A new `SecureStorageError` class plus
`SECURE_STORAGE_GENERIC_MESSAGE` / `SECURE_STORAGE_ACCESS_DENIED_MESSAGE`
constants land in `utils/errors.ts:288-298`.

## What looks good

- The wrapper pattern `runSecureStorageOperation` is the right shape
  for this concern — single ingress for all four methods, no duplicated
  try/catch, and the rethrow specifically *preserves* a
  `SecureStorageError` if the inner call already threw one
  (keychainService.ts:269-272). That's important because
  `getKeychainOrThrow` itself throws `SecureStorageError` for the "no
  keychain available" case (line 209), and we don't want to wrap-then-
  unwrap with a different message.
- The new test cluster (`keychainService.test.ts:330-371`) builds a
  reusable `expectSecureStorageFailure` helper and runs it against all
  four methods. Crucially, the assertion includes a *negative* check
  on `sensitiveFragments = ['account=acc1', '/Users/secret',
  'AccessDenied']` (lines 75-79) — i.e. the test would fail if a
  future "helpful" PR tried to surface the original message back
  through. That's the correct way to lock in a privacy invariant.
- Updating the existing
  `should return true (via fallback), log sanitized error, ...` test
  (test:152-184) to assert the *new* one-arg `debugLogger.debug` shape
  (`'Keychain initialization failed'` with no second argument) keeps
  the contract honest.

## Nits

1. `keychainService.ts:269` — `console.error('Secure storage operation
   failed:', error)`. This dumps the full original `error` object
   (with its message containing the very path/account info we just
   went to lengths to *not* throw) straight to stderr. If the goal is
   PII-scrubbing, `console.error` is no better than `throw`; both end
   up in the user's terminal and in any CI log capture. Either:
   - use `debugLogger.debug` (matches the rest of the file's intent
     that internal failures are debug-only), **or**
   - call a PII-redacting formatter that strips paths / account
     fragments before logging.
   The current code closes the *throw* leak but keeps the *log*
   leak — same data, different channel.
2. `getKeychainOrThrow` (line 209) throws
   `SecureStorageError(SECURE_STORAGE_GENERIC_MESSAGE)`. The
   `_ACCESS_DENIED_MESSAGE` constant is added (errors.ts:290-291) but
   never used. Either wire it up to the case where macOS specifically
   reports `errSecAuthFailed` / `errSecInteractionRequired`, or drop
   the constant.
3. `runSecureStorageOperation` (line 264) is `private async` and
   currently has no JSDoc. A two-line comment ("never log raw
   `error`; never throw a non-Generic message; preserve any pre-thrown
   `SecureStorageError`") would lock in the three invariants the
   surrounding tests are quietly enforcing.
4. The `verifyKeychainFunctionality` method (lines 245-258) is now
   triple-wrapped — each of the three internal calls
   (`setPassword`, `getPassword`, `deletePassword`) goes through
   `runSecureStorageOperation`. That means a single underlying error
   produces three `console.error` lines on stderr (one per failed
   step) before the function returns `false`. Either short-circuit on
   the first failure (early return) or move the wrapping out one
   level so the *whole* verification is one operation, one log line.

## Risk

Low. The error type is a subclass of `Error`, so any caller doing
`catch (e: any) { console.log(e.message) }` keeps working — they just
see the generic message instead of the leaky one. The only behavior
risk is the three-failure-log situation in #4 above; that's a noise
issue, not a correctness issue.

The privacy improvement is real and the regression coverage
(`expectSecureStorageFailure` helper) is the kind of test that
prevents future drift. Worth landing once nit #1 is addressed —
otherwise the PR's own goal is partially undermined by its own
`console.error`.
