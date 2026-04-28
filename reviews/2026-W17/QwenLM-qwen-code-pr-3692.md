# QwenLM/qwen-code PR #3692 — fix(core): route countSessionMessages through parseLineTolerant

- URL: https://github.com/QwenLM/qwen-code/pull/3692
- Head SHA: d4211c2592ef
- Files: `packages/core/src/services/sessionService.ts`, `packages/core/src/utils/jsonl-utils.ts`, `packages/core/src/utils/jsonl-utils.test.ts`
- Verdict: **merge-as-is**

## Context

The JSONL chat log format intermittently produces `}{`-glued
lines under the #3606 corruption shape (interrupted append where
two records end up on the same physical line without a separating
newline). The `read()` and `readLines()` paths already handle
this via `parseLineTolerant`, which uses
`_recoverObjectsFromLine` to extract balanced top-level objects
from a glued line. But `countSessionMessages` had its own
inline `try { JSON.parse(trimmed) } catch { continue; }` block,
so glued lines counted as zero messages instead of the
recoverable two.

## Design analysis

Two coordinated edits:

1. `jsonl-utils.ts:113`: `parseLineTolerant` is changed from
   `function` to `export function`. The docstring above it is
   updated from "Mirrors the silent skip in
   `countSessionMessages`" to "Use this from any streaming
   reader that walks JSONL line-by-line and wants the same
   recovery semantics as `read()` / `readLines()`."

2. `sessionService.ts:313-336`: the inline `try/catch` in
   `countSessionMessages` is replaced with
   `for (const record of jsonl.parseLineTolerant<ChatRecord>(trimmed, filePath)) { ... }`.
   The deduplication logic (`uniqueUuids.add(record.uuid)`) is
   preserved verbatim.

The behavioral net change: a glued line that contains two
distinct user/assistant records with distinct UUIDs now
contributes two entries to the unique set instead of zero. For
clean lines the result is identical. For permanently malformed
lines (`parseLineTolerant` returns `[]`) the loop body simply
doesn't execute — same as the old `continue;`.

The new tests in `jsonl-utils.test.ts:85-103` lock in the
contract:

- Well-formed line → `[parsed]`
- `}{`-glued line → `[obj1, obj2]`
- Unrecoverable garbage → `[]`

Each is a one-liner that documents the public surface of the
exported helper, which is exactly what you want for a small
utility being promoted from private to public.

## Risks / suggestions

1. The function visibility change from private to public is now
   a stable surface. Worth noting in any release notes /
   changelog so downstream consumers know they can rely on it.
2. `_recoverObjectsFromLine` is the actual brain here — it must
   correctly identify balanced top-level `{...}` boundaries in
   the presence of nested braces inside string values. The PR
   doesn't touch that function, so the assumption is that the
   existing test coverage is already sufficient. A brief
   confirmation in the PR description would put a reviewer at
   ease without re-reading the recovery logic.
3. Performance is identical: each line still goes through one
   `JSON.parse` attempt in the happy path (inside
   `parseLineTolerant`), and only invokes the recovery
   machinery on the rare malformed line.

## What I learned

When two readers walk the same on-disk format, they must share
the same parsing primitive — otherwise one reader silently
under-counts whatever the other recovers. The original docstring
even called this out ("Mirrors the silent skip in
`countSessionMessages`") which is the smell that there were two
copies of the parsing logic. Promoting the helper to public and
deleting the inline copy is the textbook resolution: one place
to tighten recovery, one place to test.
