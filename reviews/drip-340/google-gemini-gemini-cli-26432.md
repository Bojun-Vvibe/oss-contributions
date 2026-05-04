# Review: google-gemini/gemini-cli #26432 — feat(cli): Improve error messages for authentication failures

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26432
- **Author**: itzzSPcoder
- **Base**: `main`
- **Head SHA**: `8e643b900dbb9b56d1882b487f8dd0a3f195169a`
- **Size**: +62 / −14 across 4 files
- **Closes**: #3074

## Scope

Replaces vague auth-failure errors and stack traces with a structured, actionable message ("Reason: …" + "Fix:" with provider-specific guidance) and unifies the non-interactive auth failure exit path through `handleError` so every output mode (text / JSON / stream-JSON) reports the same fatal-auth code.

## Substantive review

### `packages/cli/src/utils/errors.ts` (+55 / −3)

- New `formatAuthError(error, authType?)` (line ~64): builds a 3-block message ("❌ Authentication Failed", "Reason: <sanitized>", "Fix: <auth-type-specific>"). Reasonable shape.
- The `sanitizedReason` regex `/[\x00-\x1F\x7F-\x9F]/g` (line ~80) strips C0/C1 control characters but explicitly preserves `\n`, `\r`, `\t`. Good defense against terminal-injection via crafted upstream error messages — note the comment correctly calls out the goal.
- The `fix` branch (line ~85) only distinguishes `google-cloud` vs everything else. There are several other `AuthType` values (Vertex AI, Workspace, etc.) — please switch on `AuthType` explicitly with a `default` case rather than a single ternary, otherwise users on other auth types get OAuth/API-key-only guidance which may not apply.
- The `fix` branch hard-codes `gemini auth login` — please confirm the exact CLI subcommand exists in this repo (some forks use `gemini login` or `gemini auth`). A wrong fix-up command in an error message is worse than no fix-up command.
- `isAuthRelatedError(error, customErrorCode?)` (line ~95) checks `customErrorCode === ExitCodes.FATAL_AUTHENTICATION_ERROR`, then the imported `isAuthenticationError(error)` predicate, then a string-match heuristic on the error message looking for "api key" + ("not valid" | "missing" | "invalid"). The heuristic is **fragile** — non-English upstream errors won't match, and the substring search will false-positive on errors that mention "api key" in passing. Consider relying on the structured predicate only and treating the string-match as a debug-only fallback.
- `handleError` (line ~118) now overrides `errorMessage` when `isAuthError` is true, then reuses the same code path for JSON/STREAM_JSON/text output. The text branch (line ~155) wraps the message in `new Error(errorMessage)` for `formatter.formatError` — fine, but please verify `formatError` does not double-prefix the "❌ Authentication Failed" header (i.e. that the formatter doesn't itself add an "Error:" line).
- The new `else { if (isAuthError) { console.error(...); process.exit(...) } throw error; }` block at the bottom of the text branch (line ~168) preserves the legacy `throw` for non-auth errors — backward compatible. ✓

### `packages/cli/src/validateNonInterActiveAuth.ts` (+5 / −11)

- The diff collapses the previous `if JSON … else debugLogger + process.exit` branching into a single `handleError(...)` call. This is the right unification — `handleError` now knows how to format and exit for every output mode. Net code shrinkage and no behaviour loss for the JSON path.
- For the text path: the previous code called `debugLogger.error(...)` and `await runExitCleanup()` (the async variant). The new path goes through `handleError` which calls `runSyncCleanup()` (sync). Please confirm this doesn't drop async cleanup steps (telemetry flush, file handle close) that the async cleanup performed.

### `.github/workflows/gemini-{automated,scheduled}-issue-triage.yml` (+1 each)

- Adds `GEMINI_CLI_TRUST_WORKSPACE: 'true'` env var to the `run-gemini-cli` step in two workflows. **Concern**: this is unrelated to the PR's stated purpose (auth error messages). It also opts the triage workflows into trusting their workspace, which has security implications for a workflow whose comment says `GITHUB_TOKEN: ''` precisely *because* it runs on untrusted inputs. These two lines should either be removed from this PR or split into a dedicated PR with a security justification.

## Verdict

**merge-after-nits** — the auth-error UX work is solid and fixes a real friction point. Three concrete asks before merge:
1. Replace the ternary in `formatAuthError`'s `fix` block with an exhaustive `switch (authType)` covering all `AuthType` values.
2. Verify the `gemini auth login` subcommand string is correct, or use a generic "Run `gemini --help` for auth options".
3. Drop the two unrelated workflow changes (`GEMINI_CLI_TRUST_WORKSPACE: 'true'`) — they don't belong in this PR and need their own security review.
4. Confirm `runSyncCleanup` covers everything the previous `await runExitCleanup` did in the non-interactive auth-failure path.
