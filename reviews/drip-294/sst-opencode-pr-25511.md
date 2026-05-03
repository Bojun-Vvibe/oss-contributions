# sst/opencode PR #25511 — fix(session): detect empty tool args and prevent abort loop

- **Repo:** sst/opencode
- **PR:** #25511
- **Head SHA:** `af294e5f56419b0be95fa05b13a42393c7c365af`
- **Author:** codeg-dev
- **Title:** fix(session): detect empty tool args from gpt-5.5 and prevent abort loop
- **Diff size:** +24 / -0 across 2 files
- **Drip:** drip-294

## Files changed

- `packages/opencode/src/session/llm.ts` (+15/-0) — adds an `isEmptyArgs` branch in the tool-call repair path that, for `apply_patch` only, rewrites the tool to `"invalid"` with a synthetic message so the model can self-correct.
- `packages/opencode/src/session/processor.ts` (+9/-0) — at processor dispatch time, aborts the tool call entirely when `value.input` is empty, calling `failToolCall(...)` with an "Empty tool arguments received" error.

## Specific observations

- `processor.ts:288-296` — the abort path is the right defensive net for the documented gpt-5.5 streaming defect. Good that it logs `{ tool: value.toolName }` so this is searchable in production. Comment "(e.g. gpt-5.5 streaming defect)" should cite the issue or upstream bug for future archeology — otherwise the next person to read this won't know whether the workaround can be removed.
- `llm.ts:354-365` — the `["apply_patch"].includes(toolName)` allow-list is suspiciously narrow. The PR description implies the empty-args defect is model-side (gpt-5.5), not tool-side, so why is `apply_patch` special? If empty args from gpt-5.5 can hit *any* tool, this allow-list will silently route every other tool to the existing fallback at `llm.ts:367` which produces an "invalid arguments" tool message. Two paths for the same defect is confusing — either widen the allow-list to all tools or document why apply_patch needs the dedicated synthetic input.
- `llm.ts:362` — the synthetic `input` is JSON-stringified, but the surrounding `failed.toolCall.input` field is typed as the raw object elsewhere in this file (e.g. the fallback at line ~373 also stringifies). Worth confirming downstream consumers tolerate string-typed input here; it's consistent with the existing fallback so likely fine.
- Both branches (`llm.ts` repair + `processor.ts` abort) will fire for the same call; the processor-side abort runs first in the normal path, which means the `llm.ts` branch is only reached when the repair pipeline gets a malformed `failed.toolCall`. Worth adding a one-line comment in `llm.ts` clarifying this layering, otherwise readers will assume one of the two is dead code.
- No test added. A unit test pinning the abort behavior for empty `{}` and the repair behavior for `apply_patch` would lock in the contract.

## Verdict: `merge-after-nits`

Correct fix and the layered defense is sensible, but: (1) cite the upstream bug, (2) justify or widen the apply_patch allow-list, (3) add at least one regression test. None block merging if the maintainer accepts the trade-off.
