---
pr: https://github.com/sst/opencode/pull/24537
sha: 4b220238
diff: +11/-1
state: OPEN
---

## Summary

Threads the original tool-call `params` (or `args`) into every tool's result object across the eight built-in tools (`apply_patch`, `bash`, `edit`, `glob`, `grep`, `lsp`, `read`, `write`) so downstream consumers that re-render a completed tool invocation can recover the inputs. Targets crash #24529 where the renderer assumed `args` was always present on tool results.

## Specific observations

- The change is mechanically uniform: each tool's returned object gets one new key (`args: params` or, for `lsp.ts`, `args` since the local already binds `args`). Eight files, one line each, no behavior change inside the tools themselves. Low risk on the tool side.
- `packages/opencode/src/tool/bash.ts:601-615` is the only non-trivial site — bash previously did `return yield* run(...)` and now binds the result first: `const result = yield* run(...); return { ...result, args: params }`. The `...result` spread preserves whatever `run` returns (stdout, exitCode, metadata), and `args` is appended last, so a hypothetical `args` key inside `result` would be overwritten. That's fine today (bash's `run` doesn't emit `args`), but worth a one-line comment so a future change to `run`'s shape doesn't silently drop the original params.
- `packages/opencode/src/tool/read.ts:213-285` — `read` returns from three different code paths (truncated text, image, and the normal text path). The PR correctly adds `args: params` to all three; missing any one of them would have produced an inconsistent shape that the renderer (per #24529) is exactly the kind of consumer that breaks on. Good coverage.
- `packages/opencode/src/tool/lsp.ts:108` uses `args` (no colon) instead of `args: params`. That's because the lsp tool's outer destructure binds the input as `args` already (vs. the others using `params`). Correct, but the inconsistency is a small readability papercut — the result key is `args` in every tool now, so the source name doesn't matter, but a reader scanning the diff will pause at the one site that omits the colon.
- No test changes. The crash in #24529 is presumably reproduced by a renderer-side test, but a tool-side regression test that asserts every tool's result has an `args` key (snapshot or shape assertion) would prevent the next tool addition from forgetting this contract. With eight tools all needing the same key, this is a contract that begs to be enforced by the type system — `Tool.define` could require result to contain `args: TParams` so omission is a compile error.
- The PR doesn't update `apply_patch.ts`'s second return (the early-return path on patch parse failure) — checking the diff context, only the success-path return at `:294` is touched. If the failure path also returns to the renderer, it will still crash on missing `args`. Worth verifying there isn't a second return site in `apply_patch.ts` that needs the same treatment.

## Verdict

`merge-after-nits` — straightforward, low-risk fix for a real crash. Two follow-ups: (1) verify no early-return in `apply_patch.ts` is missed, (2) lift the contract into the `Tool.define` type signature so the next tool author can't forget. Optional: the bash spread-then-assign pattern deserves a one-line comment about ordering.

## What I learned

When a renderer-side crash is caused by a missing field, the temptation is to make the renderer defensive (`args ?? {}`). This PR takes the opposite (and correct) tack: tighten the producer contract. The right next step is to make the contract uncircumventable via types so the bug class is gone, not just patched eight times.
