# QwenLM/qwen-code #3872 — fix(core): shrink file diff session records

- PR: https://github.com/QwenLM/qwen-code/pull/3872
- Head SHA: `3ed5888adaf73235e18bd907a702f1a82d6c97c1`
- Base: `main`
- Size: +616 / -20 across 7 files
- Files (key):
  - `packages/cli/src/acp-integration/session/emitters/ToolCallEmitter.ts` (+11/-0)
  - `packages/cli/src/acp-integration/session/emitters/ToolCallEmitter.test.ts` (+35/-0)
  - `packages/cli/src/ui/components/messages/ToolMessage.tsx` (+39/-11)
  - `packages/cli/src/ui/components/messages/ToolMessage.test.tsx` (+21/-0)
  - `packages/cli/src/ui/utils/export/collect.ts` (+2/-1)
  - `packages/cli/src/ui/utils/export/collect.test.ts` (+89/-0)
  - `packages/cli/src/ui/utils/export/normalize.test.ts` (+72/-0)

## Verdict
**merge-after-nits**

## Rationale
This is the right kind of "trim the JSONL" PR — it bounds the *saved* representation of an oversized edit (full pre-content + post-content + diff could be hundreds of KB per tool call, multiplied across a long session) without touching the in-memory representation that drives the live UI. The reviewer-focus note in the PR body explicitly calls out the contract: replay/export consumers must treat saved previews as *incomplete*, not reconstruct fake full diffs from them. The implementation honors that.

The new `truncatedForSession` flag flows through three consumers, each with proportionate test coverage:

1. **ACP emitter** (`packages/cli/src/acp-integration/session/emitters/ToolCallEmitter.ts:277-285`): when replaying a saved tool result, if `truncatedForSession === true` the emitter substitutes a plain text notice (`buildTruncatedDiffPreviewText(obj)`) instead of emitting a `type: "diff"` part. This is the critical correctness piece — without this guard, an ACP client would receive a fake mini-diff built from the bounded preview content and mistake it for the real edit. The new test at `ToolCallEmitter.test.ts:236-268` pins the expected text emission ("Full diff omitted from saved session history…").

2. **Tool message UI** (`packages/cli/src/ui/components/messages/ToolMessage.tsx`): adds a saved-session preview notice ("Saved session preview only; full diff omitted from JSONL (N chars).") above the rendered diff so live users see the truncation marker. Test at `ToolMessage.test.tsx:277-294` covers it.

3. **Export normalize/collect**: the `+89` and `+72` test additions on `collect.test.ts` and `normalize.test.ts` are the largest test surface and cover the export pipeline so that exported transcripts also carry the truncation marker rather than silently appearing complete.

The split of *what gets persisted* vs *what gets replayed* is the correct seam.

## Specific lines
- `ToolCallEmitter.ts:277-285` — early-return branch on `truncatedForSession` returning a `type: "content"` text part. Correct.
- `ToolMessage.test.tsx:280-289` — exercises the in-UI notice with the `123456` length marker.
- `ToolCallEmitter.test.ts:248-254` — sets `truncatedForSession: true`, `fileDiffLength: 200000`, `fileDiffTruncated: true`. Asserts the emitted text matches verbatim.

## Nits before merging
1. **Two flags, one concept.** The new payload carries both `truncatedForSession` (boolean) and `fileDiffTruncated` (boolean) plus `fileDiffLength`. A reader has to grok which one means what. Worth a JSDoc comment on the type defining the contract: `truncatedForSession` = "this record came out of the bounded JSONL serializer, do not reconstruct"; `fileDiffTruncated` = "the diff string itself was cut". Otherwise future contributors will conflate them.
2. **No threshold visible in the diff** — the actual byte/char threshold that triggers `truncatedForSession` lives in the core `chatRecordingService.ts` change (referenced in the PR body's vitest command but not shown in the visible diff slice). Make sure the threshold is named in code (e.g. `MAX_PERSISTED_FILE_DIFF_CHARS`) and not a magic number.
3. The PR claims "+616/-20" but the visible diff is heavy on tests. Encourage author to include a one-line entry in the changelog/CHANGES file (if the repo has one) calling out the JSONL format change, since older replayers may be confused by the new flags.

None of these block merge — the design is sound and tests cover the critical paths.
