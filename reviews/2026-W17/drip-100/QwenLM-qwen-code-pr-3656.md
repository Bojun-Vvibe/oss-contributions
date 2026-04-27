# QwenLM/qwen-code#3656 — fix(core): recover from `}{` glued records on session JSONL load

- **Size**: substantial (new test file + utility refactor)
- **Issue**: #3606

## Summary

Session JSONL files were occasionally produced with two top-level records concatenated onto one physical line (`{"uuid":"a"}{"uuid":"b"}`), most likely from a missing newline write-flush ordering bug elsewhere. The previous reader treated the whole malformed line as one `JSON.parse` attempt → threw → dropped not just the glued records but every record after the corruption point, making the entire session unloadable.

This PR adds `_recoverObjectsFromLine<T>(line)` to `packages/core/src/utils/jsonl-utils.ts:47-127` which walks the line with a brace-depth counter that respects string boundaries and `\` escapes, then attempts `JSON.parse` on each balanced top-level fragment. Anything that still fails to parse is silently skipped; the caller decides whether to warn. Hooked into `read()` and `readLines()` so existing call sites recover transparently.

## Diff inspection

The brace-walker (`jsonl-utils.ts:47-127`) handles four state-machine cases correctly:
- `escape` flag eats one char then resets — correct order vs `inString` check.
- `inString` swallows `{`/`}` so glued-record-detection only fires at depth 0 outside strings.
- `c === '"'` toggles `inString` only when not escaped.
- `depth === 0` after `}` triggers `JSON.parse(line.slice(start, i+1))`.

The 132-line test file at `packages/core/src/utils/jsonl-utils.test.ts` covers:
- well-formed single object → 1 result
- `{"a":1}{"b":2}` → 2 results
- `}{` literal **inside** a string → not split (critical correctness case)
- escaped quotes `\"` inside strings → not falsely terminating
- middle fragment unparseable, surrounding objects recovered (`{"a":1}{"oops":}{"b":2}` → `[{a:1},{b:2}]`)
- garbage line → `[]`
- `read()` of clean file unchanged
- the exact #3606 corruption shape (3 clean lines + 1 glued + 1 clean) → all 4 records recovered
- `readLines(file, 3)` respects limit even when objects come from recovery
- limit-met-by-recovery within first N
- blank lines skipped

The author also explicitly documents the limitation in the JSDoc: only top-level **object** records are recovered, not arrays. Callsite audit is not in the diff but the comment promises "all existing JSONL writers in this codebase produce object records."

## Strengths

- The state machine is the textbook correct shape — escape-then-string-then-structure ordering, no off-by-one on `start = i` vs `start = -1`.
- Test coverage genuinely exercises all five interesting branches plus integration through `read`/`readLines`.
- Silent-skip for unparseable fragments is the right call inside a "recover what you can" helper — caller-side warning keeps the helper testable.
- `_resetEnsuredDirsCacheForTest()` export hygiene with the `_` prefix is the right pattern.
- Fast path is unchanged: clean files take exactly the same code path as before; the recovery branch only activates when standard `JSON.parse` would have thrown.

## Concerns

1. **No telemetry / log emission on recovery.** When this fires in production, ops will want to know "session X had N glued records recovered" so the root-cause writer bug stays visible. Adding `logger.debug({ file, recoveredCount }, "jsonl line recovered")` (debug-level only, since it's not user-facing) would prevent the bug from going underground.
2. **Top-level-objects-only limitation is enforced by convention, not assertion.** If a future writer ever produces array records, the recovery silently skips them instead of failing loud. A one-line `if (!line.trimStart().startsWith("{")) return [];` early-out plus comment would make the limitation explicit.
3. **`_recoverObjectsFromLine` is exported with a leading underscore for tests but is also reachable as the module's stable surface.** A separate `__test__` re-export or `@internal` JSDoc tag would prevent accidental adoption by other callers.
4. **No fuzz test.** The state machine begs for property-based testing (`fast-check`) — generate random sequences of well-formed JSON objects with random concatenations and assert round-trip. The existing examples cover the obvious cases but a 5-minute fuzz harness would cover the long tail.

## Verdict

**merge-as-is** — the fix is correct, tested at the unit level for the actual #3606 corruption shape, and structurally cautious (fast path unchanged, silent-skip on unrecoverable, well-documented limitation). The telemetry / fuzz / `@internal` items are improvements, not blockers.
