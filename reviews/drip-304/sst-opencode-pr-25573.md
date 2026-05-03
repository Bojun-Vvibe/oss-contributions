# sst/opencode #25573 — fix(cf-ai-gateway): route provider options through openaiCompatible key

- **PR:** https://github.com/sst/opencode/pull/25573
- **Head SHA:** `a98026011a29bc19ed5907a1d1cbb72cca7c5cd3`
- **Author:** NathanDrake2406
- **Size:** +276 / -20 across 3 files
- **Closes:** #24432

## Summary

`reasoningEffort` and workflow `variant` inputs were silently dropped for cf-ai-gateway models routed through `ai-gateway-provider`, because `sdkKey()` in `transform.ts` had no branch for that npm package, so provider options were keyed under the wrong namespace and the SDK ignored them. This PR adds the branch plus a model-aware reasoning-effort tier table and e2e/unit coverage.

## Specific references

- `packages/opencode/src/provider/transform.ts:41-47` — adds `case "ai-gateway-provider":` returning `"openaiCompatible"`. The inline comment explicitly notes that `@ai-sdk/openai-compatible` parses `compatibleOptions` from `openai-compatible | openaiCompatible | Unified | unified` and that the kebab-case form emits a deprecation warning, so picking the camelCase form is the right canonical choice.
- `transform.ts:434-470` — new `OPENAI_NONE_EFFORT_RELEASE_DATE = "2025-11-13"`, `OPENAI_XHIGH_EFFORT_RELEASE_DATE = "2025-12-04"` constants and `GPT5_FAMILY_RE = /(?:^|\/)gpt-5(?:[.-]|$)/`. Anchor on `^` or `/` correctly avoids the `gpt-50` / `gpt-5o` false matches the comment calls out.
- `openaiReasoningEfforts(apiId, releaseDate)` correctly returns `null` for `gpt-5-pro` (no tunable knob) and special-cases the `codex` family — matches OpenAI's published behavior.
- New tests in `test/provider/cf-ai-gateway-e2e.test.ts` and `test/provider/transform.test.ts` cover both the routing fix and the tier-gating logic.

## Concerns

- The release-date gates are hard-coded constants. If OpenAI ships a new effort tier the constant has to be bumped manually; a TODO comment pointing at where to update would help future maintainers.
- `GPT5_FAMILY_RE` uses `[.-]` to match `gpt-5.4` and `gpt-5-nano`. Worth a unit test asserting `gpt-5o` is *not* matched (the comment claims it isn't, but no negative case is shown in the highlighted hunks).

## Verdict

**merge-after-nits** — fix is correct and well-commented. Suggested nit: add a negative regex test for `gpt-5o` / `gpt-50` and a comment pointing at the release-date constants for future maintainers.
