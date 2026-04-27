# QwenLM/qwen-code PR #3648 — fix(acp): repair integration against current core API

- Link: https://github.com/QwenLM/qwen-code/pull/3648
- Head SHA: `b6060b398bc94236e4b429ef9a2fae427efa71b5`
- Size: +65 / -2 across 4 files (2 prod, 2 test)

## Summary

Two-part fix to keep the ACP integration buildable against the current `@qwen-code/core` surface. Adds explicit `export type { NotificationRecordPayload }` and `export { SESSION_TITLE_MAX_LENGTH }` named re-exports to `packages/core/src/index.ts` (alongside the existing `export *` lines, which don't re-export named-only `type` exports the way ACP needs them) and locally narrows the `record.subtype` discriminant inside `HistoryReplayer.replayRecord` to a `string | undefined` so the `'notification' | 'cron'` literal-union check compiles even when upstream `ChatRecord` discriminant types tighten.

## Specific-line citations

- `packages/cli/src/acp-integration/session/HistoryReplayer.ts:55-78`: `case 'user': { const subtype: string | undefined = record.subtype; if (subtype === 'notification' || subtype === 'cron') { … } }` wraps the case body in a block scope to declare the narrowed local; the alternative would have been a non-null assertion or `as string`, both of which would silently mask future discriminant breakage. The block-scope choice keeps the type relaxation visible at the use site.
- `packages/core/src/index.ts:140` and `:148`: explicit `export type { NotificationRecordPayload } from './services/chatRecordingService.js'` + `export { SESSION_TITLE_MAX_LENGTH } from './services/sessionService.js'`. Both are paired with existing `export * from …` lines for the same module, which is the correct shape — `export *` does re-export const values but not isolated `export type` declarations under `verbatimModuleSyntax`, so the explicit type re-export is load-bearing.
- `chatRecordingService.test.ts:436-462` + `sessionService.test.ts:909-937`: four-each regression tests assert the symbols are dynamically importable from the public module path with the correct runtime shape (`expect(typeof module.SESSION_TITLE_MAX_LENGTH).toBe('number')` + `.toBe(200)` for the const, `expect(module).toBeDefined()` for the type-only export). The const test pinning the literal value `200` will catch any future widening; the type-only test is necessarily weak (types erase) but documents the contract.

## Verdict

**merge-as-is**

## Rationale

This is a maintenance fix surfacing the exact correct minimum: two missing named re-exports plus one local discriminant relaxation, no behavior change, and tests that pin the regression's blast radius. The decision to widen to `string | undefined` locally (rather than fighting the upstream `ChatRecord.subtype` type) is the right choice because the ACP layer is a *consumer* of `ChatRecord`, not the discriminant authority — pushing the literal-union back upstream would be the wrong layer.

The only thing I'd add as a non-blocking follow-up is a comment on the `subtype` local that says *why* the relaxation exists (e.g. "core's ChatRecord union may not yet have 'cron' in the discriminant; treat as string until upstream lands"), so a future reader doesn't tighten it back. But the current code is correct and the test coverage is appropriate for a compile-fix PR.

