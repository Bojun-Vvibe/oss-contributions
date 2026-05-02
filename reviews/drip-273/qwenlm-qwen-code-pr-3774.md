# QwenLM/qwen-code PR #3774 — feat(core): enforce prior read before Edit / WriteFile mutates a file

- **PR**: https://github.com/QwenLM/qwen-code/pull/3774
- **Head SHA**: `303b6b7d4ac7706d78fc3f4e40c6bef222fa7c9d`
- **Size**: +677 / −6 across 10 files
- **Verdict**: **merge-after-nits**

## What changed

A new safety invariant: `Edit` and `WriteFile` (overwrite) refuse to mutate a file unless the model has read its current contents in this session.

- New module `packages/core/src/tools/priorReadEnforcement.ts` (+116) — presumably exposes a check that consults `FileReadCache` and returns either OK or an `EDIT_REQUIRES_PRIOR_READ`-flavored error.
- `tools/edit.ts` (+30 / 0) wires the enforcement into the edit pipeline; new tests in `edit.test.ts` (+252) include a dedicated `describe('prior-read enforcement', …)` block (lines 940+ of the diff) with cases like "rejects an edit when the file has not been read in this session" — the file remains untouched and the error type is `ToolErrorType.EDIT_REQUIRES_PRIOR_READ`.
- `tools/write-file.ts` (+41 / 0) gets the same gating; `write-file.test.ts` (+155) adds matching coverage.
- `tools/read-file.ts` (+9 / −2) and `read-file.test.ts` (+36) update so a successful Read records the cache state that satisfies the new invariant.
- `tools/tool-error.ts` (+11) declares the new `ToolErrorType.EDIT_REQUIRES_PRIOR_READ` and probably its message template.
- `services/fileReadCache.ts` (+16 / −1) — `recordWrite` now seeds `lastReadAt`, `lastReadWasFull`, `lastReadCacheable` when the entry is brand-new, so a `create→edit→edit` chain (where the first Write creates the entry) doesn't fail the second Edit's prior-read check. The accompanying test (`fileReadCache.test.ts:213`) flips from "does not set lastReadAt" to "seeds read metadata when recording a write on a brand-new entry" and asserts the trio is set, with `lastReadAt === lastWriteAt`.
- All existing edit tests (line 270 onward in `edit.test.ts`) gain a `seedPriorRead(filePath)` call after their `fs.writeFileSync(filePath, …)` setup. A new helper at line 105 documents the pattern: tests that exercise *Edit business logic* (diffing, encoding, replace_all) seed the cache to bypass the new gate; the new `prior-read enforcement` describe block is the only place that intentionally omits it.

## Why this matters

Editing a file the model hasn't read is a known foot-gun: the model will hallucinate `old_string` based on its training prior or a stale view, the `old_string` may match by coincidence, and the user ends up with a quietly corrupted file. Worse, the model often "knows" what should be there and edits accordingly, but the actual content drift means the edit lands in the wrong place or destroys recent human edits. Forcing a Read before Edit/Write is a cheap, well-precedented invariant (Anthropic's Claude Code, openai/codex, and others enforce something similar). The fileReadCache `recordWrite` seeding is the subtle but critical detail — without it, the create-then-edit-then-edit pattern (where the model authored the file) would be incorrectly rejected on the second edit.

## Specific call-outs

- `recordWrite` seeding (`fileReadCache.ts:131–142`): the inline doc comment is excellent and explicitly names the failure mode it prevents ("a create→edit→edit chain would be rejected on the second edit because lastReadAt/lastReadWasFull/lastReadCacheable would still be undefined"). The test now asserts `entry.lastReadAt === entry.lastWriteAt` to lock that exact relationship — good.
- The `seedPriorRead(filePath)` helper at `edit.test.ts:105` is the right way to retrofit ~20 existing tests; the doc comment explains why each call site needs it. **Nit**: consider hoisting this helper into a small `test-utils` module shared with `write-file.test.ts` so both files use the same seeding pattern. Otherwise drift between two copies is easy.
- `EDIT_REQUIRES_PRIOR_READ` error message asserted by the test: `/has not been fully read in this session/`. **Nit**: this error message will be surfaced to the model as a tool error — make sure the actual string explicitly tells the model "call ReadFile on this path first, then retry Edit." Without that hint, weaker models may loop trying to Edit with different `old_string` values.
- Test asserts `expect(fs.readFileSync(filePath, 'utf8')).toBe('untouched content')` after rejection — confirming this is *gating*, not *warning*. That's the correct semantic but should be called out in the user-facing changelog: this is a behavior change, and any agent prompt or workflow that bypassed Read will now break.
- The PR touches 10 files but the scope is coherent: cache infra → enforcement helper → two tool wirings → error type → tests for each layer. Reasonable size.
- **Nit**: I don't see the diff for `priorReadEnforcement.ts` itself (only its filename). Reviewers should specifically scrutinize: how is "file has been read fully" defined for very large files where `lastReadWasFull` is false because the model only saw a slice? If a partial Read can satisfy the gate, models may slice-read 10 bytes then edit blindly. Conversely, if a partial Read does *not* satisfy the gate, files larger than the per-call read budget become un-editable. The `lastReadWasFull` field name suggests the strict reading; please confirm and document explicitly.
- Reading the cache test diff more carefully: `lastReadCacheable: true` is set by `recordWrite`. Make sure no downstream consumer of `lastReadCacheable` interprets a write-seeded entry as "we cached the read content" — if any code uses that flag to short-circuit a real ReadFile and serve from cache, it would serve nothing/wrong content because the write didn't actually populate any cached *bytes*.

## Verdict rationale

Solid, well-tested introduction of a useful safety invariant, with the create→edit→edit subtlety correctly handled in `recordWrite`. Two real concerns: (1) confirm partial-read semantics for `lastReadWasFull`, and (2) audit `lastReadCacheable=true` consumers to ensure write-seeding doesn't accidentally enable bytes-from-cache reads. Plus minor nits about helper sharing and error-message wording. None are blockers if the partial-read story is documented; I'd want to see that one resolved in-PR before merge.
