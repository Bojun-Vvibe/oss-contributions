---
pr: 24544
repo: sst/opencode
sha: 1c450b625cba98a083fd8516812ecf0d6df6e42d
verdict: merge-as-is
date: 2026-04-27
---

# sst/opencode #24544 — fix(session): compare message positions instead of IDs in SessionPrompt.run

- **Author**: alfredocristofano
- **Head SHA**: 1c450b625cba98a083fd8516812ecf0d6df6e42d
- **Link**: https://github.com/sst/opencode/pull/24544
- **Size**: +19/-6 across `packages/opencode/src/session/prompt.ts` only.

## Scope

Closes #23490. The session prompt loop was using lexicographic message-ID comparison (`lastUser.id < lastAssistant.id`) to decide both the loop-exit condition and the "skip messages we've already processed" filter. Custom message IDs (e.g. host-supplied IDs from external clients) break that ordering — the strings sort however they sort, regardless of actual transcript order. Fix replaces both string comparisons with array-index comparisons on the same `msgs` array that's already being scanned.

## Specific findings

- `prompt.ts:1322-1346` — three new index trackers (`lastUserIndex`, `lastAssistantIndex`, `lastFinishedIndex`) initialized to `-1` and updated atomically alongside the existing `lastUser`/`lastAssistant`/`lastFinished` captures inside the reverse-scan loop. The three branches each guard with `!lastUser` / `!lastAssistant` / `!lastFinished` so the index captured is always the latest occurrence (matching how the message refs are captured).
- `prompt.ts:1366` — the loop-exit predicate flips from `lastUser.id < lastAssistant.id` to `lastUserIndex < lastAssistantIndex`. Since the loop scans backwards from `msgs.length - 1`, `lastUser` is the **most-recent** user message and `lastAssistant` is the **most-recent** assistant message; "user came before assistant" → lastUserIndex < lastAssistantIndex is correct semantics.
- `prompt.ts:1471-1474` — the `for...of msgs` + `m.info.id <= lastFinished.id` skip is replaced by `for (let i = lastFinishedIndex + 1; i < msgs.length; i++)`. This is both correct (positions are monotonic in `msgs`) and slightly faster (no per-iteration comparison + skip).
- `-1` sentinel is safe because the comparison `lastUserIndex < lastAssistantIndex` is gated above by `lastAssistant?.finish` and `lastUser` truthiness checks at `:1359-1362` (not shown in diff but inferable from the surrounding code path) — so reaching the comparison line implies both indices got assigned.
- Single-file, single-concern change; no test added but the fix is local and the failure mode (custom ID with non-monotonic lex order causing infinite loop or incorrect message replay) is hard to harness without scaffolding a custom-ID generator. The argument by inspection is convincing.

## Risks

Low. The two changed comparisons were pure ordering-by-position, which is exactly what positions encode. The only subtle worry is if any other code path was relying on the old `lastUser.id <= lastFinished.id` comparison's specific behaviour for *equal* IDs — but `<=` only matched when the same message was both the last-finished assistant and a user, which is impossible since they're distinct roles.

## Verdict

**merge-as-is** — surgical position-vs-ID fix on a real bug that breaks any client supplying custom message IDs. The change is mechanical, the predicate semantics are preserved, and the slight perf improvement on the second loop is a free bonus. A regression test would be nice but isn't a merge blocker for a 19-line bug-fix that's clearly correct by inspection.
