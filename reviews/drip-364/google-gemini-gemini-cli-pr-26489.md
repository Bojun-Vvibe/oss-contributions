# google-gemini/gemini-cli #26489 — perf(context): skip O(N) calculateTokenBreakdown when tracer is disabled

- Head SHA: `acfe282e48e9ab8f36d3374bb021a21c148411bb`
- Diff: +12 / −6 across `packages/core/src/context/graph/render.ts` and `packages/core/src/context/tracer.ts`

## Verdict: `merge-as-is`

## What it does

Adds a public `get isEnabled()` accessor on `ContextTracer` (`tracer.ts:46-48`) that exposes the existing private `enabled` field, and uses it in `render()` to short-circuit two `tracer.logEvent('Render', ...)` calls — `'Estimation Calibration'` and `'Protection Audit'` — that *otherwise* unconditionally invoke `env.tokenCalculator.calculateTokenBreakdown(nodes)` and `Object.fromEntries(protectionReasons)` respectively (`render.ts:51-58`). Both of those argument-side-computation calls are O(N) in the size of the rendered context graph, and both are pure waste when the tracer is disabled because `logEvent` itself is already a no-op in that case — but JavaScript evaluates the argument-object before the function call, so the work runs anyway.

## Why this is good as-is

1. **Real perf bug, not a microbenchmark.** `calculateTokenBreakdown` walks every node in the context graph and runs the active tokenizer over each one's content; for a 1M-token compressed thread that's a non-trivial walk happening on every `render()` invocation, which is on the hot path during model-turn assembly. The tracer is off by default in production CLI usage, so 99% of users were paying this cost for a log line that was never emitted. The branch + accessor has the same cost as the old code in tracer-on mode and saves the entire breakdown call in tracer-off mode — strict win.
2. **Symmetric: both expensive `logEvent` arg-sites get the same gate** (`render.ts:51-58`). The PR author noticed both, didn't fix one and forget the other. The third `tracer.logEvent` further down that handles `currentTokens > maxTokens` is left alone — that's correct because its argument is a small object literal, not an O(N) computation.
3. **`get isEnabled()` is the minimal viable public surface.** Returns the existing private `enabled: boolean` directly (`tracer.ts:46-48`), no defensive copy needed, no semantic divergence from the current `if (!this.enabled) return;` guards inside `logEvent` itself. Other call sites that want the same gate can adopt it without ceremony.
4. **No behavior change when tracer *is* enabled.** Same two events fire with identical payloads; the only observable difference is "events don't fire when tracer is off, and the work to compute their payloads doesn't happen either." That matches the documented contract of a tracer.

## Optional follow-ups (not blockers)

- The same `if (tracer.isEnabled)` pattern likely applies to other call sites that compute non-trivial `logEvent` payloads — worth a one-time grep across `packages/core/src/**/*.ts` for `tracer.logEvent\(.*\b(calculate|compute|fromEntries|map|filter|reduce)\b` and gating any matches found. Strictly out of scope for this PR.
- A future refactor could lazily evaluate `logEvent` payloads via a callback (`tracer.logEvent('Render', 'X', () => ({ breakdown: env.tokenCalculator.calculateTokenBreakdown(nodes) }))`), letting `logEvent` itself decide whether to invoke. That's a bigger API change and not what this PR is for; the explicit-gate pattern here is fine.
- The PR is also currently a strict subset of the gated logic in #26490 (the same `tracer.isEnabled` accessor + the same two render-site gates appear there too, bundled with unrelated MCP discovery). #26490 should rebase to drop these lines once #26489 lands, or the maintainer should land #26489 first and force #26490 to rebase.

## Citations

- `packages/core/src/context/tracer.ts:46-48` — new `get isEnabled()` accessor
- `packages/core/src/context/graph/render.ts:51-59` — gated `'Estimation Calibration'` + `'Protection Audit'` logEvent block
- `packages/core/src/context/graph/render.ts:50` (existing) — `tracer.logEvent('Render', 'budget', ...)` left ungated because its payload is a literal object, not an O(N) computation
