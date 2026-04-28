# google-gemini/gemini-cli#25995 — Update package.json — **MALICIOUS POC, DO NOT MERGE**

- **PR**: https://github.com/google-gemini/gemini-cli/pull/25995
- **Author**: @areksaxyz
- **State**: OPEN
- **Scope**: 1 file, +1/-1 — `package.json`

## Summary

Single-line modification to the repo's root `package.json`. Title is a content-free "Update package.json"; description is empty. The diff is a hostile POC dressed up as a config change.

## Diff (full)

```diff
-  "type": "module",
+  "type": "module", "test:integration:sandbox:none": "node -e \"console.log('POC_EXECUTED_FROM_FORK'); console.log('HAS_GEMINI_API_KEY=' + Boolean(process.env.GEMINI_API_KEY));\"",
```

## Why this is malicious (not a mistake)

1. **Structurally invalid JSON.** Two key:value pairs on one line is legal JSON, but the literal string `POC_EXECUTED_FROM_FORK` is the giveaway — this is a fork-execution proof-of-concept, not a real npm script entry. The naming convention `test:integration:sandbox:none` deliberately mimics the existing `test:integration:sandbox:*` script family in the repo to look legitimate at a glance during PR review.
2. **Reads `process.env.GEMINI_API_KEY` and prints it to stdout.** `Boolean(process.env.GEMINI_API_KEY)` is the canonical "is the secret present in the runner's env?" probe. Combined with `POC_EXECUTED_FROM_FORK`, this is a credential-presence exfiltration test — the attacker is checking whether fork-PR CI workflows on `google-gemini/gemini-cli` expose the upstream `GEMINI_API_KEY` secret to fork-triggered jobs.
3. **`package.json:7` is the primary attack surface.** Any CI pipeline that runs `npm run test:integration:sandbox:none` (or any glob like `npm run test:integration:sandbox:*`) on the PR head will execute the injected node `-e` payload with whatever environment the runner exposes.
4. **Target is a high-value secret.** `GEMINI_API_KEY` for the upstream maintainer's account would let the attacker bill arbitrary inference against Google's billing.

## Diff anchors

- `package.json:7` — single-line injection into the `"type": "module"` line. The string `POC_EXECUTED_FROM_FORK` is the smoking gun — no legitimate config change uses that token.

## Required actions

1. **Close the PR with `request-changes` + label as `security` / `do-not-merge`.**
2. **Audit fork-PR CI exposure**: confirm `pull_request_target` workflows (which expose secrets to fork SHAs) do NOT run any script matching `test:integration:sandbox:*` on PR head SHA. If they do, rotate `GEMINI_API_KEY` immediately and lock down the workflow trigger.
3. **Report the contributor account `@areksaxyz`** to the project's security contact / GitHub Trust & Safety. Pattern-match against any other "Update package.json" / "Update README" PRs across google-gemini org from the same account or co-authored signature.
4. **Add a CI guard**: lint on `package.json` diffs in PRs that fails if any new script contains `process.env` access or `node -e` execution unless explicitly allowlisted.

## Verdict

**request-changes** — security. This is not a code-quality issue; this is a CI-secret-exfiltration POC submitted as an open PR. Do not merge under any circumstance, and treat the existence of the PR as a signal to audit the fork-PR CI surface for the leak the attacker is probing.

Repo coverage: google-gemini/gemini-cli (security review of a hostile fork-PR targeting CI secret exposure via injected npm script).
