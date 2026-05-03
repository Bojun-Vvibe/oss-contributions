# Review: sst/opencode PR #25549 — feat(provider): add Featherless AI provider

- **Head SHA:** `91ac99432b96208a1681bbf8686e6cd88929ab63`
- **State:** OPEN
- **Size:** +1632 / -0, 10 files
- **Verdict:** `merge-after-nits`

## Summary
Adds a Featherless.ai provider with a client-side concurrency gate to avoid 429s
from Featherless's cost-reservation billing. Plumbing follows the existing
`custom()` registration pattern (mirrors `anthropic`/`openai` shape), with a
self-contained SDK under `packages/opencode/src/provider/sdk/featherless/`.

## What works
- `provider.ts:154-174` keeps the core touch tiny (~23 lines) and only forwards
  to `createFeatherlessFetch` from `sdk/featherless/index.ts:1-96`. Clean
  separation — easy to delete later if Featherless dies.
- The release-on-stream-close wrapper at
  `sdk/featherless/stream-release.ts:1-57` correctly routes natural completion,
  consumer cancel, and read error through a single idempotent release path.
  Tests at `test/provider/featherless/stream-release.test.ts:1-107` cover all
  three.
- Cold-start snapshot fetch in `gate.ts` (called from the `start()` path the PR
  description references) plugs the hole the smoke test caught — releasing on
  headers-received would over-admit.
- 40 unit tests in `concurrency-cache.test.ts` (271 lines) and `gate.test.ts`
  (377 lines) cover FIFO ordering, SSE drain, reconnect, idempotent release,
  and corrupt-cache handling. Strong coverage for a 580-line gate+cache pair.

## Nits / issues to address before merge

1. **Bundled seed will rot.** `sdk/featherless/seed.ts:1-53` ships 11 hard-coded
   model entries that bypass models.dev. The PR body promises "the seed goes
   away once a models.dev PR lands," but no follow-up issue is linked. Add a
   `// TODO(#NNN): remove once models.dev ships Featherless` so this doesn't
   become permanent dead weight.

2. **Smoke script writes to real account.**
   `script/featherless-smoke.ts:1-171` is well-structured but unguarded — a
   stray CI run with `FEATHERLESS_API_KEY` set will spend real credits. Add a
   `--confirm` flag or `FEATHERLESS_SMOKE_CONFIRM=1` env gate.

3. **`fetchAccountState` and `lookupCosts` in the smoke script
   (`script/featherless-smoke.ts:23-44`) hard-code
   `https://api.featherless.ai/...`** — the SDK should expose a single base URL
   so dev-mode swaps are possible. Minor; could be a follow-up.

4. **Synthetic config-provider injection.** The `registerFeatherlessSeed` call
   in `provider.ts` is invoked unconditionally inside the `custom()` factory.
   Verify this doesn't double-register on hot reload; the existing config
   registry should be idempotent but worth confirming with a regression test.

5. The `LARGE_MODEL` constant in `featherless-smoke.ts:17` is a DeepSeek model
   name — fine, but consider naming it `LARGE_MODEL_DEFAULT` and allowing
   override via env, so users on plans without DeepSeek access can still smoke
   the gate.

## Verdict rationale
Surgical core change, strong test coverage, and the design notes in the PR
body show the author thought through the cold-start + release-timing edge
cases before catching them in the smoke run. The bundled seed is the only
real concern, and only as a maintenance hazard rather than a correctness bug.
Approve once a tracking issue or clear TODO marker is added for seed removal.
