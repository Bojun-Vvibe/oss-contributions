# QwenLM/qwen-code #3727 — chore(core): drop tool token usage tracking

- **PR:** https://github.com/QwenLM/qwen-code/pull/3727
- **Head SHA:** `5e2136ca55506a4d7e97165de69e5087642007df`
- **Size:** ~+1 / −80 across 22 files — telemetry/types/loggers + many test files + UI components + docs

## Summary
Removes the `tool_token_count` (a.k.a. `toolUsePromptTokenCount`, a.k.a. metric label `type: 'tool'`) dimension from the qwen-code telemetry surface. Affects: (1) `telemetry/types.ts` `ApiResponseEvent` field deletion + the `usage_data?.toolUsePromptTokenCount` read site, (2) `telemetry/loggers.ts` `recordTokenUsageMetrics(..., type: 'tool')` call deletion, (3) `telemetry/qwen-logger/qwen-logger.ts` field omission from the JSON body, (4) `telemetry/metrics.ts` counter `type` union narrowing from `'input' | 'output' | 'thought' | 'cache' | 'tool'` → `'input' | 'output' | 'thought' | 'cache'`, (5) UI components `ModelStatsDisplay.tsx`, `SessionContext.tsx`, `computeStats.ts` strip the `tool` field from rendered totals, (6) docs/telemetry.md drops the field from the documented event schema. The change is clean — every reader and writer is updated symmetrically.

## Specific observations

1. **Source-of-truth removal at `packages/core/src/telemetry/types.ts:333,363`:**
   ```ts
   -  tool_token_count: number;
   ...
   -    this.tool_token_count = usage_data?.toolUsePromptTokenCount ?? 0;
   ```
   The `usage_data?.toolUsePromptTokenCount` is the upstream Gemini SDK field. Dropping the read means the field is no longer exposed even if the SDK continues to populate it. Symmetric — once `ApiResponseEvent` doesn't have the field, every downstream consumer either had to drop it or fail to compile.

2. **Counter type narrowing at `telemetry/metrics.ts:86-92`:**
   ```ts
       attributes: {} as {
         model: string;
   -      type: 'input' | 'output' | 'thought' | 'cache' | 'tool';
   +      type: 'input' | 'output' | 'thought' | 'cache';
       },
   ```
   This is the load-bearing public-surface change. Any external observer that subscribes to `qwen-code.token.usage` and filters by `type: 'tool'` will now see zero events. **The metric name (`qwen-code.token.usage`) is unchanged**, so this is a backwards-incompatible attribute-level change to a metric that has presumably been emitted in prior releases. External dashboards counting "tool token usage" will silently flatline post-deploy.

3. **Logger field drop at `telemetry/loggers.ts:509-513`:**
   ```ts
   -  recordTokenUsageMetrics(config, event.tool_token_count, {
   -    model: event.model,
   -    type: 'tool',
   -  });
   ```
   Five lines deleted. Since `event.tool_token_count` no longer exists on `ApiResponseEvent`, this call would have been a type error if left in. Correctly deleted.

4. **Test surface symmetric updates** — at least 12 test files have a `tool: 0` or `toolTokenCount: N` field deletion. Sample: `nonInteractiveCli.test.ts:1227`, `ModelStatsDisplay.test.tsx:94`/`110`/`124`/`134`, `SessionSummaryDisplay.test.tsx:78`, `StatsDisplay.test.tsx:106-400`, `computeStats.test.ts:312-378`, `core/turn.test.ts:457-471`, `output/json-formatter.test.ts:478-490`, `telemetry/loggers.test.ts:497-510`, `telemetry/metrics.test.ts:218-227`, `telemetry/uiTelemetry.test.ts:159-180`. None of these test files added a *new* assertion that the field is gone — they only removed the field references. So the regression test for "this field stays gone" is structural (the type wouldn't compile if it came back) rather than runtime.

5. **Snapshot files updated** at `__snapshots__/ModelStatsDisplay.test.tsx.snap` — confirming the rendered text no longer contains the "Tool" row. Good — snapshot diffing will catch any UI regression that re-introduces the row.

6. **Docs at `docs/developers/development/telemetry.md:249,315`** drop both the event-field documentation and the metric `type` enum value. Symmetric with the code change.

7. **`vscode-ide-companion/src/services/qwenSessionReader.ts`** is the last touched file — the IDE-companion package reads session state; presumably it had a field reference that's now gone. The diff truncated my view of it; worth a `gh pr diff 3727 --repo QwenLM/qwen-code | grep -A20 qwenSessionReader.ts` confirmation.

## Risks

- **External dashboards/alerts that filter `qwen-code.token.usage` by `type: 'tool'`** will silently produce zero data post-merge. No deprecation cycle, no grace period. The metric attribute is dropped in one release.
- **No release-note callout in the PR description** — the title is `chore(core): drop tool token usage tracking` and the implication is internal-only, but the metric is a public-surface telemetry contract. Operators who did *not* know the field was being removed will discover it from broken graphs.
- **No replacement metric** — if "tool token usage" was a meaningful number (it represents tokens consumed by tool-use payloads in the model context), dropping it removes observability into a real cost dimension. The PR doesn't explain why — is it because `toolUsePromptTokenCount` was always 0/null upstream and the metric was misleading? Is it because tool tokens are now folded into `input`? Either is fine but should be in the PR body.
- **Saved session JSON files** with the `tool_token_count` field will still parse fine (the field is just ignored on read), but if any restore path depended on the field's presence (e.g. `event.tool_token_count + event.input_token_count` in some aggregation), it would now be `undefined + N === NaN`. A `grep -rn tool_token_count packages/` to confirm zero remaining references would lock this in.

## Suggestions

- **(Recommended)** Add a sentence to the PR description / commit message explaining *why* — "upstream `toolUsePromptTokenCount` is always reported as 0 by current Gemini SDK versions and the metric was confusing users" or "tool tokens are now subsumed by `input` count, double-counting was inflating cost dashboards." Whichever it is, name the reason.
- **(Recommended)** Release-note the public-surface change: "Telemetry metric `qwen-code.token.usage` no longer emits events with `type=tool`. Field `tool_token_count` removed from `qwen-code.api_response` event."
- **(Recommended)** Add an explicit assertion in `telemetry/uiTelemetry.test.ts` that the rendered metric output does NOT include `'tool'` as a `type` value, to lock the absence in. Currently the deletion is type-enforced; an explicit test would survive a future refactor that loosens the type.
- **(Recommended)** Run `rg -n 'tool_token_count|toolUsePromptTokenCount' packages/` in CI to confirm zero remaining references; add as a one-off lint to prevent the field from being re-added accidentally.
- **(Optional)** Consider a one-release-cycle deprecation: emit `tool_token_count: 0` and `type: 'tool'` events for one release with a `deprecated` flag in the event metadata, then remove. Less abrupt for downstream operators.

## Verdict: `merge-after-nits`

Mechanically clean, symmetric across all readers and writers, type-safe (any re-introduction will fail compile). The only concerns are the missing rationale in the PR body, the missing release-note for what is a backwards-incompatible public-surface telemetry change, and the lack of an explicit "this field is gone" runtime assertion to complement the structural type guarantee. None of these block merge but all should be addressed before tagging.

## What I learned

Removing a telemetry field is a public-surface change even when it's labeled `chore`. The bug class is "the field was emitted to external observers, removing it silently breaks their dashboards/alerts." Type-safety guarantees the field stays gone in-repo; what it does NOT guarantee is that consumers who were filtering on it know that filter now produces zero data. The right shape is **(a)** explain why in the PR body, **(b)** release-note it as a behavioral change, **(c)** add a runtime assertion (not just type-level) that the field is gone — so future "let's add it back as a quick fix" PRs trip on it.
