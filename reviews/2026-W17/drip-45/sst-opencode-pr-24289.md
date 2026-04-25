# sst/opencode #24289 — fix: Repair truncated JSON tool inputs in LLM session

- **PR:** https://github.com/sst/opencode/pull/24289
- **Head SHA:** `af4461804e89796349439f8af248774e7ce95612`
- **Files changed:** 1 — `packages/opencode/src/session/llm.ts` (+90 lines)

## Summary

Adds a best-effort `repairTruncatedJson(input)` helper and wires it into the existing `experimental_repairToolCall` fallback path. When a model emits a truncated JSON object as a tool-call argument, the repair attempts up to 10 fix-up passes: close unterminated strings, balance `{}` and `[]`, drop trailing commas, and if all else fails, drop everything after the last top-level comma.

## Line-level call-outs

- `packages/opencode/src/session/llm.ts:10-17` — `isJsonStringEscaped` walks backwards counting `\`. Correct for the standard "is this quote escaped" check. Minor: this is O(n) per quote scan and the outer loops re-scan the whole buffer on every attempt → worst case O(n² · 10) per repair. For typical tool-call payloads (a few KB) it's fine; flag for any future >100 KB tool inputs.
- `packages/opencode/src/session/llm.ts:19-22` — the `startsWith("{")` guard is correct but skips JSON arrays. A truncated `[ {...}, {...` array argument would not be repaired. Most providers wrap tool args in an object, so low-risk, but worth a one-line `// only object-shaped tool args are repaired` comment.
- `:35-41` — when `inString` is true and the last char is `\`, the code strips the trailing backslash then appends `"`. This is sound for trailing escape sequences but does not handle a partial unicode escape like `\u00`, which would still parse-fail on the next iteration; the loop will eventually hit the brace-balance branch and may produce something non-equivalent to intent. Acceptable for "best-effort" but worth documenting.
- `:43-57` — re-walks the string each attempt to count braces/brackets. This is the right approach (the buffer mutates between iterations) but suggests caching `inString` transitions in a single pass rather than two passes per attempt; trivial perf optimisation if anyone cares.
- `:59-66` — the order is "close `]` before `}`" which is correct because arrays nest inside objects in the common case, but if the truncation happened mid-`{ "k": [ "x"` the loop would close `]` first, then `}`, producing `{ "k": [ "x"]}` — that's valid and probably what the model meant. Good.
- `:74-87` — the "drop everything after the last top-level comma" branch is the riskiest: it silently discards a partial key/value pair. Only triggers if brace/bracket balancing already failed, so it's a last resort, but worth logging *which* branch produced the repair so debugging a wrong tool call later is possible.
- `:99-119` — the call-site logs `original` and `repaired` at `info` level. For long tool inputs (file contents, etc.) this can blow up logs; consider truncating both to e.g. 2 KB or moving to `debug` level. Also, on success the repaired payload is returned, but there's no telemetry counter — a counter `llm.toolcall.repair{result=success|failure}` would be very useful for tracking how often models truncate.
- Missing: no unit tests added. Given this is parser logic with several branches, even a small table-driven test (`{"a": "unterm`, `{"a": [1, 2`, `{"a": 1,`) would catch regressions cheaply.

## Verdict

**request-changes**

## Rationale

Concept is correct and addresses a real provider-side flake. Two blockers: (1) **no tests** for a non-trivial parser — this is exactly the kind of code that grows subtle bugs during the next refactor; (2) **unbounded logging** of `original` and `repaired` at `info` is a foot-gun for any user with large tool payloads. Add a small repair-table test plus log-truncation, and this becomes merge-after-nits. The array-input gap and missing telemetry counter are nice-to-haves.
