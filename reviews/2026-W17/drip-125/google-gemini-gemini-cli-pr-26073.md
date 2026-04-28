# google-gemini/gemini-cli #26073 — Fix remaining issues with generalist profile

- **PR**: [google-gemini/gemini-cli#26073](https://github.com/google-gemini/gemini-cli/pull/26073)
- **Head SHA**: `0438c09b`
- **Closes**: #26072

## Summary

Multi-part follow-up to a prior generalist-profile refactor. Three
load-bearing pieces:

1. `ContextManager.assemble()` now passes its rendered history through
   a new `hardenHistory(...)` step (defined in `historyHardening.ts`)
   before returning to the caller. The hardener appends a
   `"Please continue."` user-role sentinel turn whenever history would
   otherwise end on a model turn — closing the gap where the LLM saw
   its own last turn as the "expected response" slot rather than as
   prior context.
2. `ContextGraphBuilder` is rewritten from instance-state-bearing
   (`this.episodes`, `this.currentEpisode`, `this.pendingCallParts`,
   `this.pendingCallPartsWithoutId`) to a *pure* `processHistory(history)
   -> ConcreteNode[]` method that materializes its working state in
   locals on each call. The `clear()` method (and the `getNodes()`
   accumulator pattern) is removed. Mapper at `mapper.ts:32-40` now just
   returns `this.builder.processHistory(event.payload)` directly — the
   prior `CLEAR` / `SYNC_FULL` event-type branches that drove the
   `clear()` calls become no-ops because every call rebuilds from
   scratch.
3. `pendingCallPartsWithoutId` (the legacy fallback for tool-call/
   tool-response pairing when an `id` field was missing) is *removed*
   entirely. `parseToolResponses` now logs a structured `MISSING_CALL`
   error via `debugLogger.error` when an unmatched response arrives;
   `parseModelParts` no longer collects ID-less function-call parts at
   all.

## Specific findings

- `packages/core/src/context/contextManager.ts:175-195` — `assemble()` now
  builds a per-content `summary` array of `{role, calls, responses}`
  triples and emits it via `debugLogger.log` before calling
  `hardenHistory(finalHistory)`. The diagnostic logging is verbose
  (full JSON pretty-print of every assembled history call) — this is
  the kind of thing that should be gated behind a finer log level than
  generic `log` because every LLM request now stamps the assembled
  history into the debug log. Either gate it on a `verbose` flag or
  demote to `trace` so existing debug consumers don't drown.
- `packages/core/src/context/graph/toGraph.ts:43-118` — the
  `ContextGraphBuilder` rewrite from stateful to functional is the
  big architectural piece. Three things to call out:
  - The constructor at `:46-49` keeps `tokenCalculator` and
    `nodeIdentityMap` as instance fields (correct — those are
    *configuration*, not per-call state). All the actually-mutating
    state (`episodes`, `currentEpisode`, `pendingCallParts`) is now
    function-local at `:52-54`. This eliminates a class of bugs where
    a missed `clear()` would let prior-call state leak into the next
    call's graph.
  - `mapper.ts:32-40` is reduced to just
    `return this.builder.processHistory(event.payload)` — the prior
    `if (event.type === 'CLEAR') { ... return [] }` and
    `if (event.type === 'SYNC_FULL') { this.builder.clear() }`
    branches are gone. Effectively, `CLEAR` events now flow through
    `processHistory` (which builds an empty node list from the empty
    payload) and `SYNC_FULL` events trigger a full rebuild because
    every call is a full rebuild now. Behaviorally consistent — but
    `CLEAR` semantically means "drop state", and now it emits a
    nodes-list (possibly empty, possibly not depending on payload)
    instead of an empty array. Worth a one-line comment naming why
    the CLEAR branch is safe to remove.
  - `getNodes()` is gone. The combined "finalize active episode if
    complete + return all episodes' nodes flattened" logic moved
    inline into the bottom of `processHistory` at
    `:114-122`. The `nodes` variable used at `:122` (`Generated
    ${nodes.length} nodes from ${copy.length} episodes`) is referenced
    but the diff shows it being computed in code I can't see in the
    truncated hunk — assuming it's just `copy.flatMap(ep =>
    ep.concreteNodes)` collapsed inline.
- `toGraph.ts:144-156` — `parseToolResponses` now logs a structured
  `MISSING_CALL` error on every unmatched function-response, with the
  `id` and `name` named in the log line. This is good observability
  but should *also* surface to telemetry, not just debug log: an
  unmatched response is a real correctness issue (the model called a
  tool with an ID we never recorded) and silently logging it to debug
  hides that. A `tracer.logEvent('GraphBuilder', 'unmatched_response',
  ...)` call would put it in the same observability surface the rest
  of the context manager uses.
- `toGraph.ts:243-251` — `parseModelParts` simplification: function-call
  parts without an `id` field are now silently *dropped* (no log at
  all on this path). Prior code went into the `pendingCallPartsWithoutId`
  list-with-name-matching dance to retro-pair them. The new behavior is
  correct policy (ID-less function calls from non-conformant providers
  shouldn't be silently round-tripped through the context graph because
  the `MISSING_CALL` log on the response side won't even show up — there
  was no call recorded), but it should *also* log: a model emitting an
  ID-less function call is a provider-conformance issue worth surfacing.
- `contextManager.barrier.test.ts:73-84` — the existing barrier test
  is updated for the new `Please continue.` sentinel: previous version
  asserted `lastModel = contentNodes[length-1]` and `lastUser =
  contentNodes[length-2]`; new version inserts a sentinel check at
  `[length-1]` ("user role with text 'Please continue.'") and the
  real last user/model land at `[length-3]` and `[length-2]`. This
  is the right test update for the new contract — it pins the
  hardener invariant ("history that ends on model gets a sentinel
  turn appended") at the same test that pins the barrier invariant.

## Nits / what's missing

- The `debugLogger.log(...)` summary in `contextManager.ts:185-194` is
  going to be very high-volume. Should be gated on a verbose level or
  a per-component flag.
- `MISSING_CALL` on the response side and ID-less call drop on the
  model side should both surface to the telemetry layer, not just
  debug log. These are correctness signals.
- The `if (!behavior)` guard in `fromGraph.ts:51-54` (`NO BEHAVIOR
  FOUND for node type: ...`) is good defense-in-depth, but
  `behavior.serialize(node, writer)` was previously trusted to always
  succeed. Adding the guard is correct, but a missing-behavior should
  fail loudly in tests (e.g. assert in dev builds) rather than just
  silently `continue` in prod — a missing serializer means the node's
  contribution is dropped entirely from the rendered history, which
  could be load-bearing context the model needs.
- The `hardenHistory` function itself isn't shown in the diff (it's
  imported from `./historyHardening`). The PR title says "remaining
  issues with generalist profile" so I'm assuming a prior PR landed
  the function. Worth checking that the hardener idempotency invariant
  is pinned ("calling `hardenHistory(hardenHistory(h))` returns the
  same result as `hardenHistory(h)`") — without that, repeated
  assemble calls could append multiple sentinels.
- The `ContextGraphBuilder` rewrite is a substantial architectural
  shift (stateful → pure). PR body doesn't call this out as a
  contract change for any external consumer of the builder class. If
  any caller outside this PR's diff was relying on `builder.clear()`
  or `builder.getNodes()` existing as public API, they're broken.
  Worth a quick `rg "builder\.(clear|getNodes)"` to confirm scope.
- The `contextManager.barrier.test.ts` update has a comment "The
  HistoryHardener appends a 'Please continue.' user turn if we end
  on model" which embeds the sentinel string literal in the test
  comment. Should be exported from the hardener module as a constant
  so a future text change updates one place.

## Verdict

**merge-after-nits** — the three pieces are individually well-motivated
(the hardener closes a real history-shape bug, the builder rewrite
removes a class of state-leak bugs by construction, and dropping the
ID-less function-call fallback is the right policy decision for
provider conformance), but the diagnostic-logging volume needs to be
gated, the unmatched-call signals should reach telemetry not just
debug log, the missing-behavior guard in `fromGraph.ts` should fail
loudly in dev, and the `ContextGraphBuilder` API removal needs a quick
external-caller sweep before merge.
