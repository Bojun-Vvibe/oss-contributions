# sst/opencode PR #25066 — fix(provider): auto-enable interleaved reasoning for Kimi K2.6 models

- **URL:** https://github.com/sst/opencode/pull/25066
- **Head SHA:** `b0229fee90decd33947549e470023eb97d94bf19`
- **Diff size:** +75 / -1 across 3 files
- **Verdict:** `merge-as-is`

## Specific diff references

- `packages/opencode/src/provider/provider.ts:1201-1207` — the one-line behavior change: the existing DeepSeek auto-detect for `interleaved.reasoning_content` is widened with `|| apiID.includes("kimi-k2")`. The placement is inside the same `!existingModel && apiNpm === "@ai-sdk/openai-compatible"` guard, so this only fires for *custom* Kimi entries that don't already have a registry definition — exactly the surface that was reported as broken (Moonshot returning "thinking is enabled but reasoning_content is missing"). Won't override an explicit `model.interleaved` value or an `existingModel.capabilities.interleaved`, which is the right precedence order.
- `packages/opencode/test/provider/provider.test.ts:315-353` — new test asserts that both `kimi-k2.6` and `kimi-k2.5` declared under a `@ai-sdk/openai-compatible` custom provider get `{ field: "reasoning_content" }` automatically. The fixture covers the two relevant model IDs (the substring match `kimi-k2` would also catch K2.7+ once Moonshot ships them, which is a deliberate forward-compat decision worth noting in the PR body but not a blocker).
- `packages/opencode/test/tool/fixtures/models-api.json:11547-11561` and `:64112-64126` — adds the missing `kimi-k2.6` registry entries for both the `opencode` (Zen) and `opencode-go` providers with `interleaved: { field: "reasoning_content" }` baked in, plus realistic cost/limit blocks (`input: 0.32`, `output: 1.34`, `cache_read: 0.054`, 262144 ctx / 65536 out). These mirror the shape of the adjacent `kimi-k2.5` entries, so reviewers can diff them visually.

## Reasoning

This is a textbook small-surface provider patch: a single conditional widened from `deepseek` to `deepseek || kimi-k2`, mechanically backed by a unit test that follows the exact pattern of the existing DeepSeek test in the same file. The bug class is well-understood — Moonshot's API enforces that assistant messages in multi-turn tool-use sequences carry the `reasoning_content` field, and the `interleaved` capability is the flag that tells `ProviderTransform.message()` to preserve it. Without the patch, every custom Kimi-K2 entry running through `@ai-sdk/openai-compatible` would silently fail on round 2.

The substring match on `kimi-k2` is appropriately scoped: it's gated by `apiNpm === "@ai-sdk/openai-compatible"` and only applies when there's no existing registry entry, so a non-K2 Kimi model (or a K2 served through a different SDK that doesn't need this field) is unaffected. The forward-compat behavior — K2.7, K2.8 etc. will inherit the same default — is the desired outcome based on Moonshot's stated API contract and the three issues this PR closes (#20427, #24722, #23646).

The fixture additions to `models-api.json` are large in line count but mechanical; both the OSS-repo (`opencode`) and Go-repo (`opencode-go`) provider sections get the same K2.6 entry, keeping the fixture coherent across the two transports. No public API changes, no migration, no flag flip — and the fact that the existing DeepSeek case already shipped this pattern in production means the regression risk on the broader registry is essentially zero. Ship it.
