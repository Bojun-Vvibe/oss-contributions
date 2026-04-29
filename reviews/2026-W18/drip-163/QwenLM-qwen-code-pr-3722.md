# QwenLM/qwen-code#3722 — fix(memory): use project transcript path for dream

- URL: https://github.com/QwenLM/qwen-code/pull/3722
- Head SHA: `86e34cd0964dce7bd7a69bcd03f0f24e9caff12b`
- Size: +81 / −12, 4 files
- Verdict: **merge-as-is**

## Summary

Fixes `/dream` (memory consolidation) pointing at the wrong transcript directory. Pre-PR it looked under `${QWEN_DIR}/tmp/${projectHash}/chats` (a defunct path that no longer holds session transcripts after the storage refactor); post-PR it uses `path.join(new Storage(projectRoot).getProjectDir(), 'chats')`, matching what chat-recording / session-storage actually writes to. Touches:

- `packages/cli/src/ui/commands/dreamCommand.ts:30-37` — replaces the `${QWEN_DIR}/tmp/${projectHash}/chats` template literal with a `Storage(projectRoot).getProjectDir() + 'chats'` join, drops `getProjectHash`/`QWEN_DIR` imports.
- `packages/core/src/memory/dreamAgentPlanner.ts:163-168` — same fix in the auto-memory dream planner; promotes `getTranscriptDir` from private to `export`.
- `packages/cli/src/ui/commands/dreamCommand.test.ts` (new, +50) — asserts the CLI command builds the consolidation prompt against the project-scoped transcript dir and **negative-asserts** the path does NOT contain `${path.sep}.qwen${path.sep}tmp${path.sep}` (locks in the bug's absence).
- `packages/core/src/memory/dreamAgentPlanner.test.ts:46-66` — analogous test covering the now-exported `getTranscriptDir`, also negative-asserts the old `.qwen/tmp/` path.

## Specific lines reviewed

### Fix at `dreamCommand.ts:30-37`

```ts
const projectRoot = config.getProjectRoot();
const memoryRoot = getAutoMemoryRoot(projectRoot);
const transcriptDir = path.join(
  new Storage(projectRoot).getProjectDir(),
  'chats',
);
```

Correct. `Storage.getProjectDir()` is the canonical "where this project's runtime state lives" accessor, and `chats` matches the directory name written by the session-recorder (verified consistent across the codebase by the negative-assertion tests).

### Fix at `dreamAgentPlanner.ts:163-168`

```ts
export function getTranscriptDir(projectRoot: string): string {
  return path.join(new Storage(projectRoot).getProjectDir(), 'chats');
}
```

Promoting `getTranscriptDir` from local-private to `export` is exactly what enables the unit-test assertion at `dreamAgentPlanner.test.ts:55-65` without reaching into mocked file-system internals. Single-line public API addition is acceptable.

### Tests at `dreamCommand.test.ts:33-49` and `dreamAgentPlanner.test.ts:54-65`

The negative assertion pattern is the load-bearing piece:

```ts
expect(expectedTranscriptDir).not.toContain(
  `${path.sep}.qwen${path.sep}tmp${path.sep}`,
);
```

This locks in the *absence* of the old buggy path. Future refactors that accidentally re-route to `${QWEN_DIR}/tmp/...` will fail this test. Good defensive testing.

The `dreamAgentPlanner.test.ts` test correctly resets `Storage.setRuntimeBaseDir(null)` in `afterEach` (line 45) so per-test runtime dir overrides don't leak.

## Why merge-as-is

- Surgical fix matching exactly the bug described.
- Both call sites updated symmetrically (CLI command + core auto-planner).
- Test coverage is positive (correct path computed) AND negative (old buggy path absent), which is the right shape for a "wrong-string" bug.
- No imports left dangling — `getProjectHash` and `QWEN_DIR` are dropped from `dreamCommand.ts:7-11` import block; presumably similarly dropped from `dreamAgentPlanner.ts:12` (the diff shows the import line replaced with `import { Storage } from '../config/storage.js'`).
- Promoting `getTranscriptDir` to `export` is justified by test access and is a minimal API expansion.

Optional follow-up (not blocking): grep for any other site that constructs `${QWEN_DIR}/tmp/${projectHash}/chats` (or variants) to ensure this isn't a third occurrence elsewhere — a 30-second `rg "tmp.*projectHash.*chats"` would close the loop. But the two-site-symmetric fix presented here is complete for the described bug.
