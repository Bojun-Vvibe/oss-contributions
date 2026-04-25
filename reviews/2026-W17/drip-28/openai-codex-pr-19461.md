# PR #19461 — fix: Bedrock GPT-5.4 reasoning levels

- Repo: openai/codex
- Head SHA: 25fb97c046879d3d46f24522dd0a23b5176acce0
- Files touched: 1 (+54 / -32)

## Summary

The Bedrock provider's `openai.gpt-5.4-cmb` catalog entry was
derived at runtime from the bundled `gpt-5.4` model spec via
`bundled_models_response()` + `model_info_from_slug("gpt-5.4")`.
That picker exposes `xhigh` as a reasoning level — which Bedrock
rejects at request time:

```
Invalid 'reasoning': Invalid 'effort': unknown variant `xhigh`,
expected one of `high`, `low`, `medium`, `minimal`
```

This PR replaces the runtime lookup with an explicit hardcoded
`ModelInfo` literal in
`codex-rs/model-provider/src/amazon_bedrock/catalog.rs:38-89`
(`gpt_5_4_cmb_bedrock_model`) that pins
`supported_reasoning_levels` to exactly the Bedrock-supported set
via the new `gpt_5_4_cmb_reasoning_levels()` helper at
`amazon_bedrock/catalog.rs:114-122` (Minimal, Low, Medium, High
— deliberately no XHigh).

## Specific findings

- The replaced symbol `bundled_gpt_5_4_model()` (deleted at
  `catalog.rs:48-60` in the old version) was a soft lookup with
  a `model_info_from_slug` fallback; that compounded two failure
  modes — wrong reasoning-level set AND inconsistent context
  window if the bundled spec drifted. Hardcoding the literal
  removes both.
- `GPT_5_4_CONTEXT_WINDOW = 272_000` and
  `GPT_5_4_MAX_CONTEXT_WINDOW = 1_000_000` are pinned at
  `catalog.rs:16-17`. These constants now diverge from whatever
  `bundled_models_response()` says for the same model elsewhere
  in the codebase. That's fine for this PR (intentional carve-
  out) but worth a comment block flagging that the Bedrock CMB
  catalog deliberately diverges from bundled metadata.
- The renamed `bedrock_oss_model` helper at
  `catalog.rs:74-107` (was `bedrock_model`) is purely cosmetic
  — collapses the shared GPT-OSS construction without behavior
  change. Worth doing in the same PR; keeps the surface
  readable.
- Test rewrite at `catalog.rs:155-167`: old test
  `gpt_5_4_cmb_uses_gpt_5_4_spec` asserted equality against the
  bundled spec (which is exactly the bug). New test
  `gpt_5_4_cmb_advertises_only_bedrock_supported_reasoning_levels`
  asserts only the field that matters. Correct narrowing — the
  old assertion would have prevented this fix from landing.

## Risks / nits

- The literal divergence from bundled metadata is a
  maintenance hazard once the upstream model spec changes
  (e.g. the bundled `gpt-5.4` adds new tool support flags).
  A `// keep in sync with bundled gpt-5.4 except
  supported_reasoning_levels` comment + a regression test that
  diff-asserts the *intersection* of the two would make the
  drift visible.
- The Bedrock validation error string in the PR body says
  "expected one of `high`, `low`, `medium`, `minimal`" — that
  matches the four literals in `gpt_5_4_cmb_reasoning_levels`.
  No `auto` / `default` level slipped through. Good.
- Other Bedrock model entries (`openai.gpt-oss-120b`,
  `openai.gpt-oss-20b`) are constructed by the
  `bedrock_oss_model` helper and don't go through the bundled
  lookup, so they're not affected. Worth a one-liner in the
  PR body confirming GPT OSS reasoning levels remain
  `Stage::Stable` per their existing test.

## Verdict

**merge-after-nits** — Tight, single-file fix with the right
shape (replace soft runtime lookup with an explicit literal,
narrow the regression test to what matters). Add the drift-
warning comment + a small intersection-with-bundled-spec test
before landing.

## What I learned

When a runtime catalog lookup of a *bundled* model spec is
producing the wrong shape for a *deployed* surface (Bedrock
strictly enforces a smaller `effort` enum than the OpenAI
canonical), the right fix is to stop sharing the spec — not
to filter it after the fact. Tests that assert structural
equality against the source-of-truth catalog will block the
fix from landing; the test has to narrow to the field that
the deployment surface actually constrains.
