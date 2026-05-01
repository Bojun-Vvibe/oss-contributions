# QwenLM/qwen-code #3774 — feat(core): enforce prior read before Edit / WriteFile mutates a file

- **Repo:** QwenLM/qwen-code
- **PR:** https://github.com/QwenLM/qwen-code/pull/3774
- **HEAD SHA:** `303b6b7`
- **Author:** wenshao
- **Verdict:** `merge-after-nits`

## What the diff does

Closes a long-standing "model edits from imagination, not evidence" failure mode by gating `EditTool` and `WriteFile` mutations on prior, full, cacheable Read of the same file in the same session. Built on the session-scoped `FileReadCache` that PR #3717 introduced — that PR was framed as a perf optimization for repeat reads, but the cache was always a means to *this* end: making the model produce a `read` before being permitted to produce an `old_string` against the file's bytes.

The new gate lives at `packages/core/src/tools/priorReadEnforcement.ts:1-116` (new file) — a single async function `checkPriorRead(cache, filePath, verb)` returning a structured `PriorReadDecision` (`ok: true` | `ok: false` with `type: ToolErrorType`, `rawMessage` for the model, `displayMessage` for the user). The decision has three branches:

- File doesn't exist on disk → `ok: true` (new-file creation is exempt; nothing to read first).
- `cache.check(stats).state === 'fresh'` AND `lastReadAt !== undefined` AND `lastReadWasFull` AND `lastReadCacheable` → `ok: true`. **Load-bearing detail**: the conjunction of `lastReadWasFull && lastReadCacheable` is what closes the partial-read attack — a model that called `read_file` with `offset`/`limit`/`pages` has only seen a window, so its `old_string` is still potentially imagined for the unseen region. `lastReadCacheable` excludes auto-memory partials.
- `state === 'stale'` → `FILE_CHANGED_SINCE_READ` with prose telling the model to re-read.
- `state === 'unknown'` OR partial/non-cacheable fresh → `EDIT_REQUIRES_PRIOR_READ` with prose telling the model to read fully (without offset/limit/pages).

Wired into `edit.ts:152-183` *before any disk read happens* — important because the prior-read check must close the read-less content oracle (the `NO_OCCURRENCE_FOUND`/`EXPECTED_OCCURRENCE_MISMATCH`/`NO_CHANGE` error codes leak whether a guessed `old_string` exists in the file). New-file creation is exempt at `:158` via the `fileExists` guard; opt-out via `getFileReadCacheDisabled()` is preserved at `:158` for users who don't want the gate.

The author also fixed a corollary at `read-file.ts:673,680`: auto-memory reads now ALWAYS go through `recordRead` even though they skip the file-unchanged fast-path, because *not* recording them would cause prior-read enforcement on a file the model just legitimately read to fail. The comment at `priorReadEnforcement.ts:55-65` ("when a tool *creates* a file via Edit `old_string === ''` or WriteFile new path, the FileReadCache `recordWrite` call seeds `lastReadAt` so a subsequent edit on that same file passes here") shows the same "writes count as reads" property is preserved end-to-end.

Test surface at `edit.test.ts:225-451` covers `prior-read enforcement`, `exempts new-file creation` (`:358`), and `bypasses enforcement entirely when fileReadCacheDisabled is true` (`:429`); plus `read-file.test.ts:709-745` locks the auto-memory recordRead behavior.

## Why it's right

- **Reframes `FileReadCache` as the source of truth for "has the model seen this file's current bytes."** The error codes EditTool emits leak content via cardinality, so the gate must run *before* any content-derived comparison; placing it at `edit.ts:158` rather than inside `calculateEdit` is correct.
- **Conjunction of `lastReadWasFull && lastReadCacheable`** closes the partial-read class of the bug. A purely `lastReadAt !== undefined` gate would let a model paginate to bypass.
- **Stale vs unknown branches with distinct error codes** lets the model self-correct: `FILE_CHANGED_SINCE_READ` and `EDIT_REQUIRES_PRIOR_READ` give it actionable next-tool-call hints, and the prose at `:88-91,107-110` explicitly tells it to re-issue `read_file` "without offset / limit / pages."
- **Auto-memory reads now `recordRead` unconditionally** — without that fix, the gate would over-fire on a workflow that does `read_file <auto-memory-file>` → `edit_file <same-file>`, which is a common pattern.
- **`getFileReadCacheDisabled()` opt-out preserves operator escape valve** — the gate is a strict additive default; users on legacy workflows can disable it.
- **The exemption for `old_string === ''` (new file via Edit) and WriteFile-to-new-path** is named in the doc comment at `priorReadEnforcement.ts:55-65` rather than buried in code — load-bearing comment, because a future "tighten the gate" attempt would otherwise break new-file creation.

## Nits

1. **`stat` failure → `ok: true`** at `priorReadEnforcement.ts:71-75` swallows error class. `ENOENT` correctly means "new file, no prior read needed," but `EACCES`/`EPERM` should NOT silently be treated as "ok to edit" — the model can't see the file, but it's also presumably going to fail to write to it; bubbling EACCES would be a clearer signal than letting the edit attempt and fail with a confusing downstream error.
2. **`displayMessage` strings are not localized** — `displayMessage: \`${ToolNames.READ_FILE} required before ${verbDisplay}.\`` and the `'editing this file'`/`'overwriting this file'` strings are hard-English. If qwen-code has an i18n surface, these should route through it.
3. **No test for the partial-read bypass attempt** — `read_file(path, offset=0, limit=10)` followed by `edit_file(path, ...)` should hit `EDIT_REQUIRES_PRIOR_READ` because `lastReadWasFull === false`. The visible test at `edit.test.ts:225-451` covers no-read-at-all and stale, but not partial. This is the property the conjunction was designed to enforce; without a test, a future "let's loosen `lastReadWasFull`" change wouldn't be caught.
4. **`PriorReadVerb` is `'editing' | 'overwriting'`** — a typed union with two cases is fine, but the `verbDisplay` derivation at `:106` reaches into a separate vocabulary (`'editing this file' | 'overwriting this file'`). If a third tool ever calls `checkPriorRead`, two places will need updating. A single `interface VerbProse { progressive: string; gerund: string }` per verb would centralize.
5. **The `getFileReadCacheDisabled()` opt-out** is a global kill-switch; a per-tool opt-out (e.g. for a "fast unsafe edit" mode the user explicitly enables) would let safety-conscious users keep the gate for `WriteFile` while disabling it for `Edit`. Out of scope for this PR.
6. **`ToolErrorType.EDIT_REQUIRES_PRIOR_READ`** is the new error code — confirm downstream telemetry/metrics dashboards count it as a recoverable model-side error rather than a user-side one, so dashboards don't suddenly spike "errors" on rollout.

`merge-after-nits` — right reframe (FileReadCache as the source of truth for "has the model seen this file"), correct conjunction at the gate (`lastReadWasFull && lastReadCacheable` closes partial-read bypass), exemptions named in comments rather than buried in code, auto-memory `recordRead` corollary fixed in the same PR. Wants the partial-read regression test, EACCES handling distinction, and i18n routing before merge.
