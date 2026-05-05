# google-gemini/gemini-cli #26540 — fix(core): resolve policy engine bugs affecting tool approvals

- URL: https://github.com/google-gemini/gemini-cli/pull/26540
- Head SHA: `11eadac9affc67f6a10c95d0d4af3d2419b4d525`
- Diff: +23 / -13 across 4 files (`policy/policy-engine.ts` +8/-6, `policy/utils.ts` +2/-2, `scheduler/policy.test.ts` +1/-1, `utils/shell-utils.ts` +12/-4)

## Findings

1. **Null-byte regex fix at `packages/core/src/policy/utils.ts:105-108`** is correct and important. `buildParamArgsPattern` was emitting `\\\\0` (which in a regex matches a literal backslash followed by `0`) as the JSON-property delimiter, but `stableStringify` actually emits literal `\x00` bytes. So every "Always Allow" pattern matching a specific argument was silently dead — the regex never matched and the user got re-prompted on every invocation. The fix to `\\x00` makes the regex match an actual NUL byte, aligning with `stableStringify`'s output. The corresponding test update at `packages/core/src/scheduler/policy.test.ts:773` confirms the contract.
2. **Redirection downgrade at `packages/core/src/policy/policy-engine.ts:288-300`** correctly tightens the policy. Previously, `YOLO` and `AUTO_EDIT` both bypassed the `ASK_USER` downgrade for shell redirection (`>`, `>>`, `|` to file). The new behaviour: YOLO unconditionally bypasses (consistent with YOLO's "trust everything" semantics), but AUTO_EDIT only bypasses when sandboxing is enabled (`!(this.sandboxManager instanceof NoopSandboxManager)`). This closes a real escalation path — `AUTO_EDIT` mode previously allowed an agent to write arbitrary files outside the workspace via `echo malicious > /etc/something` without ever prompting, even with no sandbox.
3. **Shell wrapper enhancement at `packages/core/src/utils/shell-utils.ts:809-832`** adds a second pattern (`scriptPattern = /^\s*(?:(?:\S+\/)?(?:sh|bash|zsh))\s+([a-zA-Z0-9_\-./]+\.sh)\s*$/i`) that strips `bash script.sh` to just `script.sh`, allowing approval rules keyed on the script path to match. Correct intent. **Concern**: the pattern requires the `.sh` suffix and rejects any args after the script path. Real-world invocations like `bash ./deploy.sh prod` or `sh /path/with space/script.sh` won't match. Consider widening to allow trailing args (and matching only on the script-path group), or document the limitation.
4. **Concern — character class**: `[a-zA-Z0-9_\-./]+` excludes spaces, tildes, `~`, and most non-ASCII characters. A script at `~/scripts/foo.sh` (literal tilde, before shell expansion) won't match. Acceptable trade-off for a security-sensitive matcher (overly broad would be worse), but worth a comment explaining the deliberate restrictiveness.
5. PR description claims tests exist; the diff only updates one existing test assertion. New tests for the AUTO_EDIT-with-sandbox branch and the new `scriptPattern` would harden the change. Validation checklist mentions running `policy-engine.test.ts` and `shell-utils.test.ts` — but if no new test cases are added, those runs only verify regressions, not the new behaviour.

## Verdict

**Verdict:** merge-after-nits
