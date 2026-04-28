# sst/opencode PR #24724 — regenerate go and zen tables from models.dev

- **PR**: https://github.com/sst/opencode/pull/24724
- **Author**: @KTibow (Kendell)
- **Head SHA**: `7e60df514714d7544dae788474aac3f62cb1920e`
- **Size**: +130 / −127 across 2 docs files (`go.mdx`, `zen.mdx`)

## Summary

Pure docs sync — closes #24723. Regenerates the two pricing/usage-limit tables for the Go and Zen subscription tiers from upstream models.dev data. Notable corrections: Kimi K2.6 now shows ~3× more requests-per-period than the stale value, all `MiMo-V2-*` model display names are normalized from hyphenated to space-separated form (e.g. `MiMo-V2-Pro` → `MiMo V2 Pro`), and the API-endpoints table loses the `@ai-sdk/anthropic` and `@ai-sdk/alibaba` rows for MiniMax/Qwen because those models now go through the chat-completions endpoint with the openai-compatible adapter.

## Verdict: `merge-as-is`

Docs-only PR and the regeneration is mechanical from upstream — no code paths affected, no risk of behavioral regression. The K2.6 number correction is a real factual fix (the prior table understated by ~3×), and the request/period accuracy improvement to integer-aligned-to-1 instead of integer-aligned-to-50/250 is a genuine fidelity win.

## Specific references

- `packages/web/src/content/docs/go.mdx:65-78` — model-name normalization to `MiMo V2 Pro` etc. matches the upstream models.dev display names; consistent with how the table at `:99-110` and the endpoint table at `:155-168` now spell them. Three places, all updated symmetrically.
- `packages/web/src/content/docs/go.mdx:97-110` — Kimi K2.6 jumps from `1,150 / 2,880 / 5,750` to `3,412 / 8,531 / 17,062` requests per 5h/week/month, the largest single factual correction. Per the PR body's reference to upstream models.dev commit `aa30ce3`, this is the result of K2.6's per-token price coming down. Customers reading the prior page were being told they got ~3× *fewer* requests than reality.
- `packages/web/src/content/docs/go.mdx:153-168` — endpoint table loses two `@ai-sdk/anthropic` rows (MiniMax M2.7/M2.5) and two `@ai-sdk/alibaba` rows (Qwen3.6/Qwen3.5 Plus), both now going through the openai-compatible chat/completions path. Verified against the corresponding zen.mdx changes — both tables now uniformly use `@ai-sdk/openai-compatible` for those four models. Consistency check passes.
- `packages/web/src/content/docs/zen.mdx:62-110` — full table rewrite. The biggest change is *every* GPT-5.x and Claude row is migrated from the model-native endpoint (`/zen/v1/responses`, `/zen/v1/messages`) to the unified `/zen/v1/chat/completions` path with the `@ai-sdk/openai-compatible` adapter. This is a real product/operational decision worth highlighting in release notes — users who pinned `@ai-sdk/openai` or `@ai-sdk/anthropic` in their config will need to regenerate. Worth a one-line callout at the top of the zen page that says "as of this update, all Zen models route through the openai-compatible endpoint."

## Nits

1. The PR body says "It also updates some of the notices for correctness" but the diff doesn't show any non-table prose changes — either the body is overstating or the prose changes were reverted. If the latter, fine.
2. `packages/web/src/content/docs/go.mdx:118-122` — the per-model token-pattern descriptions ("MiMo V2 Pro — 350 input, 41,000 cached, 250 output...") still hardcode the patterns that drove the request-per-period math. If those patterns are *also* tracked in models.dev, regenerating the table without regenerating the patterns risks the table going out of sync with the math description over time. Consider auto-generating both from the same upstream source.
3. The `@ai-sdk/openai-compatible` migration noted above is the kind of operational change that benefits from a `:::caution` callout box; otherwise customers will only discover their config broke when the next request 404s.
