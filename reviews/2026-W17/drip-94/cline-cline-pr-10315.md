# cline/cline #10315 — Fix gemini flash 3 preview price

- Author: dirac-run (Max Trivedi)
- Head SHA: `63a20dcbb352d82c34b639eecef364d6a8c22e8c`
- Single file: `src/shared/api.ts` (+4 / −16)

## Specifics

- Two parallel pricing-table edits, one per provider (`vertexModels` at `api.ts:1017-1021` and `geminiModels` at `api.ts:1530-1542`). The shape change in both places: `cacheWritesPrice: 0.05` is replaced with `cacheReadsPrice: 0.05` + `cacheWritesPrice: 0`. This is correcting a field-name mix-up — Google's published pricing for gemini-flash-3-preview puts the $0.05/MTok rate on **cache reads** (i.e. context-cache hits), not cache writes (which are free for this model). Old code was charging users 0.05 on every cache write event, which over a long session with frequent cache invalidations would noticeably overstate the spend estimate displayed in the UI.
- The 13-line deletion in `geminiModels` removes a `tiers: [...]` block (`api.ts:1543-1556` in the old file) that defined two context-window tiers (200k and `Number.POSITIVE_INFINITY`) with the same `inputPrice: 0.3, outputPrice: 2.5, cacheReadsPrice: 0.03`. Removing this means the model now uses the flat `inputPrice: 0.5 / outputPrice: 3.0` from the parent struct regardless of context window. **This is a behaviour change beyond the bug-fix scope** — if Google still publishes a discounted long-context tier the displayed spend will now be too high, and if they retired the tier the change is correct. The PR title does not call this out.
- `vertexModels` does not have an analogous `tiers` deletion — only the cache field rename. So the two providers now diverge in tier handling for the same logical model, which is a maintenance smell.

## Concerns

- The `tiers` removal needs maintainer confirmation: is the tiered pricing actually retired upstream, or is this an oversight? Link to Google's pricing page in the PR description would settle it. If retired, the same removal should arguably apply to `vertexModels` for consistency. If not retired, restore the `tiers` block in `geminiModels`.
- No test or fixture update. The `api.ts` model table is plain data so a snapshot test isn't strictly needed, but a one-liner pricing-calculator unit test asserting `(input=1M tokens, output=1M tokens, cacheReadHits=1M) → expected_cost` for this model would prevent silent reintroduction of the field-name swap.
- Net `+4 / −16` for a "fix the price" PR signals scope creep — the bug-fix portion is `+2 / −2`, the rest is the tier deletion.

## Verdict

`needs-discussion` — the cache-field rename is correct and merge-worthy on its own. The `tiers` removal is undocumented in the PR description and should either be split into a separate PR with a citation to the upstream pricing change or be applied symmetrically to `vertexModels`. Ask the author before merging.

