---
pr: 24435
repo: sst/opencode
sha: d2dc95437e9e57ace02bdb26d218be7aa3a151d6
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode #24435 — fix: bump openrouter sdk version to resolve deepseek reasoning issue (bug was in sdk pkg)

- **Author**: rekram1-node
- **Head SHA**: d2dc95437e9e57ace02bdb26d218be7aa3a151d6 (MERGED into `dev`)
- **Size**: +83/-4 across 4 files (`bun.lock`, `packages/opencode/package.json`, `packages/opencode/src/provider/transform.ts`, `packages/opencode/test/session/message-v2.test.ts`)
- Closes: #24261, #24190

## Scope

A two-part fix for the OpenRouter + DeepSeek reasoning round-trip that's been the subject of three other PRs this week (#24443, #24441, #24428). The author confirms the underlying defect was actually in `@openrouter/ai-sdk-provider` itself, not in opencode's transform layer:

1. Bump `@openrouter/ai-sdk-provider` from `2.5.1` → `2.8.1` (`packages/opencode/package.json:118` + `bun.lock:393`, `bun.lock:1585`).
2. Skip opencode's own `interleaved.field` rewrite when the provider is OpenRouter, because the new SDK version emits the correctly-shaped `reasoning_details` without our help (`packages/opencode/src/provider/transform.ts:193-198`).

Plus a regression test (`packages/opencode/test/session/message-v2.test.ts:876-948`) that constructs a synthetic OpenRouter `deepseek/deepseek-v4-pro` model with `capabilities.interleaved.field = "reasoning_details"`, runs an assistant message with one `reasoning` part + one `text` part through `ProviderTransform.message`, and asserts the output preserves `providerOptions.openrouter.reasoning_details` rather than collapsing it.

## Specific findings

- **The transform-layer carve-out is the right shape.** The diff at `transform.ts:193-198` adds `model.api.npm !== "@openrouter/ai-sdk-provider"` to the existing guard. That's exactly the level of granularity the bug requires: the `interleaved.field` rewrite was a workaround for SDK breakage, and now that the SDK is fixed, opencode should *not* double-process. Hardcoding the npm package name is acceptable here because `model.api.npm` is already the canonical key opencode uses to identify the SDK driver elsewhere.

- **Lock file is consistent.** Both the resolution at `bun.lock:393` (workspace top-level) and the package definition at `bun.lock:1585` are bumped. The new `ai-gateway-provider/@openrouter/ai-sdk-provider` entry pinned to `2.5.1` at `bun.lock:5723-5727` is the AI Gateway transitive consumer, which is fine — that's a different consumer with its own pin.

- **Test asserts the high-value invariant.** The test at lines 876-948 explicitly checks that the resulting assistant message content contains a `{ type: "reasoning", text: "thinking", providerOptions: { openrouter: { reasoning_details: [...] } } }` block — i.e., that opencode's transform passes `reasoning_details` through `providerOptions` *unmodified*. That's exactly the invariant the SDK-version bump is supposed to restore.

- **Coordinates with the broader DeepSeek-reasoning thrash.** Three other PRs in the same window (#24443 second-pass preservation, #24441 oa-compat token double-counting, #24411 Kilo `reasoning_details` validity) are working different facets of the same family of bugs. This PR's verdict comment from the maintainer ("bug was in sdk pkg") makes it the *root cause* fix for the OpenRouter slice; the others remain valid for non-OpenRouter providers (Kilo, oa-compat, deepseek native). Keep that mental model when reviewing the others — this one doesn't subsume them.

- **Already merged into `dev`.** Verified state. Review value here is post-hoc validation: reading the test diff, the carve-out is narrow enough that I'd have approved the same shape pre-merge.

## Risk

Very low. The carve-out is explicit (string equality on `model.api.npm`), the SDK bump is a minor (`2.5.x` → `2.8.x` within the same major), and the test enforces the new behavior. The only failure mode would be if `2.8.1` introduces an unrelated regression for non-DeepSeek OpenRouter models — that's an SDK-quality concern, not a code-shape concern, and pinning forward is the correct way to find out.

## Verdict

**merge-as-is** — already merged. The carve-out at `transform.ts:193-198` is the minimum-surface change that lets the upstream SDK fix do its job, and the regression test locks in the new behavior. The coordination with #24443 / #24441 / #24411 should be tracked in a single follow-up issue so the next person debugging "DeepSeek reasoning" through opencode has one place to land.

## What I learned

When a transform layer was originally added as a workaround for upstream SDK breakage, the *correct* fix flow is: (1) get the SDK fix landed upstream, (2) bump the dep, (3) carve out the now-redundant workaround behind a precise predicate (here: `model.api.npm` equality), (4) add a regression test that pins the new contract. Skipping step (3) leaves the workaround silently double-processing forever, which is how legacy compat code ossifies. The two-line change at `transform.ts:193-198` is a textbook example of "shrink the workaround the moment you no longer need it."
