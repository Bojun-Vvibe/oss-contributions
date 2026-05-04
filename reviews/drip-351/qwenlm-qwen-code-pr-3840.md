# QwenLM/qwen-code PR #3840

- **Title**: feat(core): refuse Edit/WriteFile when the file changed since last read
- **Author**: ihubanov
- **Head SHA**: `c6de8c171be7dc9905ffc2ea60b65a04411e3e42`
- **Diff**: +193 / -0

## Summary

Adds `FileReadCache.staleWriteReason()` — a check returning a human-readable abort reason when a file's mtime/size drifted since the model's last tracked Read or write. Wires it into Edit/WriteFile to fence the "model overwrites with stale knowledge" failure mode. Treats unknown files as fair game so brand-new file creation isn't blocked.

## File-by-file

### `packages/core/src/services/fileReadCache.ts` (L70-100)

- **L88-100 `staleWriteReason`**: delegates to existing `this.check(stats)`, gates on `state !== 'stale'`. Critical correctness detail at L91: only `'stale'` returns a reason — `'unknown'` (file the model never touched) returns null, allowing fresh file creation. This matches the docstring at L78-83.
- **L92-95 age computation**: prefers `lastReadAt` over `lastWriteAt`. Reasonable since Read is the typical pattern; the `(${ageSeconds}s ago)` annotation is genuinely useful for the model to reason about whether the change is its own (sub-second) or external.
- **L96-99 message wording**: "Re-read the file before writing — its contents are no longer what you saw." This is the right shape: imperative, actionable, model-readable.

### `packages/core/src/services/fileReadCache.test.ts` (L9-57)

Four tests cover the matrix correctly:
- **L10-13**: unknown file → null (no block on new files). Critical baseline.
- **L15-19**: fresh file → null.
- **L22-35**: mtime drift after Read → non-null, contains `/x/foo.ts` and `/re-read/i`.
- **L37-46**: size drift after Write → non-null, contains `/has been modified/i`.
- **L48-56**: the most important test — `recordWrite` then immediate `staleWriteReason` with same stats returns null. Without this, two consecutive writes by the same tool would self-block. The comment at L49-51 explains the contract.

### `packages/core/src/tools/edit.ts` (and `edit.test.ts` L113)

Test mock adds `getFileReadCacheDisabled: () => false` (L113) — implies the production wiring respects a disable flag. Need to see the edit.ts hunk to confirm the call uses `.staleWriteReason()` correctly and that disable-flag short-circuits before the check (diff was truncated past L120 in capture). Assuming standard pattern.

## Risks

- Race: between `staleWriteReason` check and the actual write, another agent could mutate the file. This is fundamental and not addressable without filesystem locks; the fence reduces but doesn't eliminate the window.
- Behavior depends on `fs.stat` returning monotonic mtimes. On some networked filesystems mtimes can go backwards or have low resolution — could yield false positives. Likely rare enough to ship.
- The "unknown file is fair game" rule means a malicious or confused tool plan could overwrite an existing file the model never Read. Comment at L80-83 explicitly endorses this trade-off, which is correct (otherwise we couldn't create new files), but worth flagging in tool docs.

## Verdict

**merge-after-nits**
