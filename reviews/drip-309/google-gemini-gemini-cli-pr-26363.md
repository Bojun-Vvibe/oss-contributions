# google-gemini/gemini-cli #26363 — Fix: ensure coreEvents listener cleanup on all exit paths in nonInteractiveCli

- PR: https://github.com/google-gemini/gemini-cli/pull/26363
- Author: Akash504-ai
- Head SHA: `171683efcc973e6103c7f189fdca9692fd29d99e`
- Updated: 2026-05-02T05:19:30Z

## Summary
Fixes a regression in the JSON-mode auth-failure path of `validateNonInteractiveAuth`. Previously, after `handleError(...)` was called, the function returned `undefined` instead of propagating the underlying error, causing downstream `nonInteractiveCli` cleanup hooks (and the JSON output writer) to behave inconsistently. The one-line fix re-throws the captured error after `handleError` runs.

## Observations
- `packages/cli/src/validateNonInterActiveAuth.ts:59`: adding `throw error;` immediately after `handleError(...)` makes the function semantics: "in JSON mode, surface the error to the caller rather than swallow it." But `handleError` (not in diff) likely calls `process.exit` or queues an exit via cleanup. If `handleError` already exits, the `throw` is dead code. If it does *not* exit (which the regression test at `validateNonInterActiveAuth.test.ts:476-489` proves — `threw` becomes true and `result` is undefined), then this throw is exactly the right fix. Either way, harmless; please confirm in the PR description which of the two cases applies so future readers don't strip the "redundant" throw.
- `packages/cli/src/validateNonInterActiveAuth.test.ts:467-489`: the regression test asserts both `threw === true` and `result === undefined`. The latter assertion is slightly misleading — when `await` throws, `result` is never assigned, so `expect(result).toBeUndefined()` is trivially true regardless of the fix. Consider replacing it with `expect(() => /* something that depends on the thrown error */).toBe(...)`, or capture and assert the error type/message: `expect(threw).toBe(true)` plus `expect(thrownError).toBeInstanceOf(Error)`.
- The PR title says "ensure coreEvents listener cleanup on all exit paths" but the diff does not touch any `coreEvents` listener registration or `removeListener`/`off` call. The actual change is "re-throw error in JSON-mode auth path." Title and body should be aligned to the actual change so future bisects/audits aren't confused.
- No coverage for the *non-JSON* branch (the `else` at line 60+) — that path already calls `runExitCleanup()` and exits, presumably unchanged. Existing tests should cover it; if not, an assertion that JSON-mode and text-mode behave consistently w.r.t. cleanup would be valuable.
- This is a tiny, targeted fix that closes a real footgun (silent `undefined` payload in JSON mode). Worth landing once title/scope is clarified.

## Verdict
`merge-after-nits`
