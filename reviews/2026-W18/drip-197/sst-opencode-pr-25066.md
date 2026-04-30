# sst/opencode PR #25066 — fix(provider): auto-enable interleaved reasoning for Kimi K2.6

- PR: https://github.com/sst/opencode/pull/25066
- Head SHA: `b0229fee90decd33947549e470023eb97d94bf19`
- Files touched: 3 (`packages/opencode/src/provider/provider.ts` +1/-1, `packages/opencode/test/provider/provider.test.ts` +40/-0, `packages/opencode/test/tool/fixtures/models-api.json` +34/-0). Net +75/-1.

## Specific citations

- The behavioral change is exactly one extended boolean expression at `packages/opencode/src/provider/provider.ts:1201-1204`: the `apiID.includes("deepseek")` sniff is widened to `(apiID.includes("deepseek") || apiID.includes("kimi-k2"))`, gated by `!existingModel && apiNpm === "@ai-sdk/openai-compatible"`. This selects the `{ field: "reasoning_content" }` interleaved capability instead of the hard-coded `false` fallback for any user-defined openai-compatible model whose id substring-matches `kimi-k2`.
- The new test at `packages/opencode/test/provider/provider.test.ts:312-352` exercises both `kimi-k2.6` and `kimi-k2.5` against a `custom-provider` configured with `npm: "@ai-sdk/openai-compatible"`, asserting `provider.models["kimi-k2.6"].capabilities.interleaved` equals `{ field: "reasoning_content" }`. The companion `custom DeepSeek openai-compatible model` test (already in the file at `:355+`) is the canonical pattern this clones.
- Two parallel insertions into `packages/opencode/test/tool/fixtures/models-api.json` (lines `:11547-11563` and `:64112-64128`) add the `kimi-k2.6` model row to two provider sections of the fixture with `interleaved: { field: "reasoning_content" }` set explicitly. Identical model rows duplicated across both fixture sections is consistent with how other Kimi entries (e.g., `kimi-k2.5` at `:11540` and `:64105`) are mirrored.

## Verdict: merge-as-is

## Concerns / nits

1. **Substring match `apiID.includes("kimi-k2")` is broader than intended.** It catches `kimi-k2.5`, `kimi-k2.6`, `kimi-k2.6-instruct`, but also any future model id like `kimi-k2-mini-non-thinking` or `not-kimi-k2-at-all-suffix`. The pattern matches the existing `apiID.includes("deepseek")` precedent so it's not a regression in style, but the prior `drip-11` review flagged a similar `id.includes("deepseek")` substring match being narrowed into a four-family allow-list — that direction (`kimi-k2.5 | kimi-k2.6 | kimi-k2-thinking`) would be more precise. Out of scope for this 1-line fix; worth a follow-up issue.
2. **The fixture row in `models-api.json` advertises `interleaved` explicitly** but the runtime sniff at provider.ts:1201 only fires when `!existingModel`. For a Kimi model that is *already* in the fixture-backed catalog, the runtime path takes `existingModel?.capabilities.interleaved` first. So the fixture row and the runtime sniff cover *different* call paths (catalog vs user-defined custom). That's the correct layering, but a one-line comment in `provider.ts:1201` clarifying "this branch only fires when the model is not in the catalog" would prevent a future reviewer from concluding the fixture entries are redundant.
3. **No negative test** that asserts `kimi-k2.5-non-thinking` (or whatever the non-reasoning Kimi K2 variant is called) is *not* given the interleaved capability when the user wants raw `<think>` blocks. If Moonshot ships a non-reasoning K2 family model, this PR will silently misclassify it. Worth a test that constructs a `kimi-k2-non-thinking` custom model with `reasoning: false` configured and asserts `interleaved` ends up `false`.
4. **Two identical 17-line JSON insertions** into the fixture are auditable but suggest the fixture should be generated from a smaller source-of-truth. Out of scope.
