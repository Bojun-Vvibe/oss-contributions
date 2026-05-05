# google-gemini/gemini-cli #26534 — fix(core): Fix chat corruption bug in context manager

- Head SHA: `e9ce4a4d2d57dee08ae246f897b6b622095284bb`
- Diff: +345 / -72 across 7 files (Fixes #26521)

## Findings

1. Two distinct fixes are bundled. **Fix A** (the headline corruption bug) is at `packages/core/src/context/contextManager.ts:58-64` — replaces the diff-based `prunePristineNodes(newIds)` + `appendPristineNodes(addedNodes)` pair with a single `syncPristineHistory(event.nodes)` call. The old code had a structural bug: `addedNodes = event.nodes.filter((n) => event.newNodes.has(n.id))` filtered the *full* node list to only newly-marked-new nodes, but then `appendPristineNodes` appended them at the end of the buffer regardless of where they actually appeared chronologically in `event.nodes` — so an upstream history reorder would corrupt the buffer's chronology. The new `syncPristineHistory` (tested at `contextWorkingBuffer.test.ts:200-340`) syncs the entire pristine slice in chronological order in one shot.
2. **Fix B** is the preview-node leakage at `contextManager.ts:248-266` and `graph/render.ts:23-103`. `render()` now accepts a `previewNodeIds: ReadonlySet<string>` parameter and filters preview nodes out at three call sites: `:30` (no-budget path), `:64` (budget-healthy path), and `:102-103` (post-management path). Without this, preview nodes (representing the in-flight pending request) would leak into the rendered LLM contents and the model would see its own future input duplicated. The targeted unit test at `render.test.ts:14-63` pins this with a 3-node fixture where `preview-1` is excluded from the rendered output.
3. The `toGraph.ts:152-160` change widens the legacy environment-header skip from "first turn only" (`turnIdx === 0`) to "any turn", and loosens the discriminator from `'This is the Gemini CLI.'` (with period) to `'This is the Gemini CLI'` (no period). The widening is defensible because legacy headers can appear mid-history after history-rewriting operations, but the discriminator change is a soft regression risk — any new product surface that emits `<session_context>` blocks containing the substring `"This is the Gemini CLI"` (in any context) will now be silently dropped. Worth a more specific anchor (e.g. require the substring to be at a known offset within the `<session_context>` envelope).
4. Two `tracer.logEvent` calls at `render.ts:62-69` are removed (the "View is within maxTokens" + "Render Context for LLM" pair) and replaced with a single `'Budget is healthy. GC Backstop bypassed.'` event. This loses the `renderedContext` payload from the trace for the budget-healthy path — slight observability regression. If anyone is reading these traces in Cloud Trace / OTel for debugging, they lose visibility into the rendered content for the most common path.

## Verdict

**Verdict:** merge-after-nits
