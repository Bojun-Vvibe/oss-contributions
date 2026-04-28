# QwenLM/qwen-code#3692 — fix(core): route countSessionMessages through parseLineTolerant

- Head SHA: `d4211c2`
- Author: qqqys
- Files: 3 / +33 / −7
- Refs #3681 (Item 1), #3606 (corruption shape)

## Summary

Fixes the discrepancy where `SessionService.countSessionMessages` returned a smaller count than `listSessions` would later show on resume, because the count path had its own readline loop with inline `JSON.parse` + silent `catch { continue }` that dropped any line failing strict parse — including the `}{`-glued lines that #3606's read-side recovery (`parseLineTolerant`) now handles. Exports `parseLineTolerant` from `jsonl-utils.ts` (was previously private to `read()` / `readLines()`) and routes the count path through it. Adds three unit tests for the newly exported helper.

## Specific observations

- Surface change at `packages/core/src/utils/jsonl-utils.ts:115` — `function parseLineTolerant<T>` → `export function parseLineTolerant<T>`. The accompanying docstring update at `:109-114` correctly reframes the function from "mirrors the silent skip in `countSessionMessages`" (now stale — the silent skip is what's being replaced) to "the streaming-reader entry point". Right ergonomic.
- Load-bearing call-site flip at `packages/core/src/services/sessionService.ts:330-336` — replaces the `try { JSON.parse(trimmed) } catch { continue }` pattern with `for (const record of jsonl.parseLineTolerant<ChatRecord>(trimmed, filePath)) { ... }`. The for-loop iteration over the recovered-records array preserves the `user`/`assistant` filter semantics on each record from a `}{`-glued line — a glued line that had `[user, assistant]` will now contribute *both* uuids to the set, where before it contributed neither. This is the literal bug being fixed.
- The PR explicitly **rejects** the issue's suggestion to call `jsonl.read()` + dedupe by uuid, with a load-bearing rationale at `:312-317` of the docstring update: `countSessionMessages` is on the `listSessions` hot path (default page size 20) and active sessions can hold thousands of records — full-loading every file just to count uuids is wasteful. Keeping the streaming readline loop preserves the original O(uuid-set) memory profile. This is the right call and worth surfacing in the PR body, which it does.
- Three-cell test matrix at `packages/core/src/utils/jsonl-utils.test.ts:85-103` covers (a) well-formed line returns `[obj]`, (b) `}{`-glued line returns both records in order, (c) unrecoverable input returns `[]`. Three cells is the correct minimum — pins the contract at both edges (well-formed and unrecoverable) plus the load-bearing recovery cell. Critically, the second cell `'{"uuid":"a"}{"uuid":"b"}'` exactly mirrors the #3606 corruption shape, so a future change to `_recoverObjectsFromLine` that silently breaks this test will surface immediately.
- Item 2 from #3681 (write-side `fsync` / atomic-rename for durability) is **explicitly deferred** with PR-body rationale citing the issue's own "Open for discussion — no immediate action required" guidance. This is the right scope discipline — bundling a write-side rewrite into a count-bug fix would have forced a much larger review surface.
- No regression test on `countSessionMessages` itself at `packages/core/src/services/sessionService.test.ts` — the PR claims `46/46 passing` for `vitest run ... sessionService.test.ts` but I don't see the new test cells listed. The two boundary cells worth adding: (a) a session file with one `}{`-glued line containing two `user` uuids returns count=2, not count=0; (b) a session file with one unrecoverable line + one good line returns count=1, not count=0. Without these the count-side fix is only indirectly verified through `parseLineTolerant`'s direct tests.

## Verdict

`merge-after-nits`

## Rationale

Right-shape minimal fix that re-uses an existing recovery primitive instead of duplicating the recovery logic in the count path. Scope discipline is good (Item 2 deferred with cited rationale). Test cells correctly pin the load-bearing recovery shape. Nits: (1) add the two `sessionService.countSessionMessages`-level cells named above so the integration is pinned end-to-end, not just at the helper layer; (2) confirm the docstring correction at `:312-317` of the production code (visible in diff) lands — without it a reader hitting `countSessionMessages` will see a streaming readline loop and not realize the recovery semantics are inherited from the helper. Both nits are low-effort and would close out the change cleanly.

