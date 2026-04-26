# QwenLM/qwen-code PR #3648 — fix(acp): repair integration against current core API

- Repo: QwenLM/qwen-code
- PR: #3648
- Head SHA: `79a0e123`
- Author: @B-A-M-N
- Diff: +63/-2 across 4 files

## What changed

Surgical ACP-integration repair: re-exports `NotificationRecordPayload` (type) and `SESSION_TITLE_MAX_LENGTH` (value) from `packages/core/src/index.ts` so the `acp-integration` package can import them after a recent core refactor narrowed the public surface, plus a defensive `case 'user': { … }`-block scope + `const subtype: string | undefined = record.subtype` aliasing in `HistoryReplayer.replayRecord` to match looser narrowing in the current `ChatRecord` type. Adds 4 regression tests asserting `NotificationRecordPayload` is importable + handles empty/special-character `displayText`.

## Specific observations

- The lexical-block addition at `packages/cli/src/acp-integration/session/HistoryReplayer.ts:56-80` correctly fixes a `case`-fall-through lint trap that would otherwise be triggered by declaring `const subtype` inside a bare `case` body. The `subtype: string | undefined` widening at line 60 is the right escape hatch for the `record.subtype === 'notification' | 'cron'` literal-narrowing problem when `ChatRecord.subtype` was widened upstream — though a cleaner long-term fix is to ask core to re-narrow via a discriminated union rather than widen the consumer.
- The two re-exports at `packages/core/src/index.ts:140` (`export type { NotificationRecordPayload } from './services/chatRecordingService.js'`) and `:148` (`export { SESSION_TITLE_MAX_LENGTH } from './services/sessionService.js'`) are the minimum-surface fix; both already had `export *` lines two lines above, so the explicit re-exports are belt-and-suspenders. That's actually defensive against future `export *` filtering and worth keeping.
- The four added tests in `chatRecordingService.test.ts:435-460` are weak — three of them just assert that `payload.displayText` round-trips through a `Record<string, unknown>` literal, which doesn't exercise the actual `NotificationRecordPayload` shape. The `module = await import(...)` assertion at line 438 only proves the module loads, not that the type is exported (TypeScript types are erased at runtime). Consider replacing with a `// @ts-expect-error` negative test plus a real `satisfies NotificationRecordPayload` positive assertion.

## Verdict

`merge-after-nits`

## Rationale

Correct integration-repair fix, low-risk, addresses a real downstream-breakage. Pre-merge: tighten the runtime tests to actually assert the type contract (or drop them and let `tsc` carry the load), and file a follow-up issue asking core to re-narrow `ChatRecord.subtype` so consumers don't need the `: string | undefined` widening trick.
