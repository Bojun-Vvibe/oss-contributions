# google-gemini/gemini-cli #26125 — fix(cli): prevent ACP stdout pollution from SessionEnd hooks

- PR: https://github.com/google-gemini/gemini-cli/pull/26125
- Head SHA: `29d9efe33a63141bae4fb3c4d37c3126c9b753f9`
- Author: cocosheng-g (Coco Sheng)
- Files: `packages/cli/src/gemini.tsx` (+4/−1), `packages/cli/src/acp/acpConsolePatcher.test.ts` (+201/−0)
- Fixes: #25345

## Observations

- Real bug: in ACP mode (Agent Client Protocol — NDJSON over stdout), the global `ConsolePatcher` redirects `console.*` calls to stderr to keep stdout clean for the protocol stream. On final cleanup, `ConsolePatcher.cleanup()` *unpatches* before `SessionEnd` hooks have finished firing. Any `console.log` from a project hook during that window writes raw text to stdout → corrupts the NDJSON stream → ACP client parser dies on a non-JSON line. This is a textbook teardown-ordering bug.
- Fix shape: in ACP mode, **don't register `ConsolePatcher.cleanup()` at all** during the final cleanup sequence. Debug logs stay redirected to stderr through process exit, NDJSON protocol integrity on stdout is maintained.
- The diff in `gemini.tsx:+4/−1` is a 4-line ACP-mode conditional gate around the cleanup registration. Right blast radius for a teardown-ordering fix — don't change anything else, just skip the unpatch in ACP mode.
- **Why "skip cleanup entirely" is the correct choice over "delay cleanup until after hooks"**: ACP processes are about to exit anyway. The `console.*` patching is held in process memory; on process exit it's reclaimed by the OS regardless of whether `cleanup()` ran. The only thing `cleanup()` does is restore the original `console.*` references *for the rest of the process lifetime* — which is zero for an ACP process at shutdown. Skipping it is functionally identical to running it correctly, with zero risk of a hook-output-during-teardown race.
- For non-ACP mode (interactive TUI), the user is human and can tolerate `console.*` messages going to stderr or stdout interchangeably during shutdown — no protocol parser is reading. So the original `cleanup()` registration is preserved for that path.
- Test surface: `acpConsolePatcher.test.ts:+201/−0` — 201-line dedicated test file. PR claims it "verifies the conditional cleanup logic" — the right cells to look for are: (a) ACP-mode positive: cleanup not registered → `console.*` still patched at simulated exit, (b) non-ACP-mode positive: cleanup *is* registered → `console.*` restored at simulated exit, (c) regression cell: `console.log` during simulated SessionEnd hook in ACP mode lands on stderr (not stdout). The third cell is the load-bearing one — it pins the **observable user-facing behavior** (clean NDJSON), not just the implementation detail (cleanup-not-registered).

## Risks / nits

- 201 lines of test for a 4-line fix is a **good** ratio for a teardown-ordering bug — these are notoriously hard to regress without dedicated cells, since process-exit semantics aren't well-modeled by most unit tests. Don't apologize for the test surface size.
- ACP-mode detection: needs to be the *same* discriminator the rest of the CLI uses (presumably an `--acp` flag check, or `process.env` sniff, or a singleton `AcpMode.isEnabled` getter). If two different ACP-mode discriminators exist in the codebase, this fix can drift. Worth checking the import.
- "Skip cleanup entirely" works only if `ConsolePatcher` truly has no other side effects beyond the `console.*` patching. If it also (e.g.) closes a log file handle, removes a tmpdir, or unregisters a process listener, those would silently leak in ACP mode. Worth grepping `ConsolePatcher.cleanup` for non-`console.*` work.
- The fix doesn't help if hooks log to stderr expecting it to be captured by the patcher — the patcher *is* the redirect, and once unpatched (in non-ACP), stderr from hooks goes to terminal. Same as today, so no regression, but worth a comment that hooks-using-stderr is fine in non-ACP because that's where the ConsolePatcher was redirecting *to* anyway.
- Long-running ACP processes: this fix is about shutdown teardown specifically. Mid-session, `ConsolePatcher` is fine. Verify the gate is on the cleanup-registration call only, not on the patching call.
- Hooks running fire-and-forget async work that logs *after* the synchronous SessionEnd handler returns: the existing patcher would have caught those (since unpatch hadn't happened yet), and now (with cleanup skipped) it still catches them. Good — no regression on the async-hook-output path.

## Verdict: `merge-after-nits`

**Rationale:** Right diagnosis (teardown-ordering: unpatch before hooks finish), right fix shape (skip cleanup-registration in ACP mode entirely — process exit reclaims the patch state anyway), right blast radius (4 lines in `gemini.tsx`), and a 201-line dedicated test file is the right ratio for a teardown bug. Nits before merge: (a) verify the ACP-mode discriminator matches the canonical one used elsewhere in the codebase, (b) confirm `ConsolePatcher.cleanup()` has no work besides `console.*` restoration that would silently leak when skipped, (c) ensure the third test cell pins observable NDJSON cleanliness end-to-end (not just the cleanup-not-registered implementation detail).
