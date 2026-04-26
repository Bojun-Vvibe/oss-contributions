# QwenLM/qwen-code #3620 — fix(core): match DeepSeek provider by model name for sglang/vllm (#3613)

- **Author:** wenshao
- **Head SHA:** `91a7964a39cf16e9f40d5a45390bbad721710e96`
- **Size:** +36 / -2 (2 files)
- **URL:** https://github.com/QwenLM/qwen-code/pull/3620

## Summary

`DeepSeekOpenAICompatibleProvider.buildRequest` already flattens
multi-part `content` arrays into joined strings to dodge a sglang
deepseek-v4 jinja template bug
(`TypeError: sequence item 0: expected str instance, list found` at
`encoding_dsv4.py:336`). But the gate that decides whether to apply that
flattening — `isDeepSeekProvider` — was matching only on
`api.deepseek.com` in the baseUrl. Anyone serving a DeepSeek model
behind sglang / vllm / ollama / a corporate gateway was silently
bypassing the workaround and hitting the original 500.

## Specific findings

- **Core change is correct.**
  `packages/core/src/core/openaiContentGenerator/provider/deepseek.ts:25-35`
  now returns true if **either** the baseUrl includes `api.deepseek.com`
  **or** the model name (case-insensitive) contains `deepseek`. The
  ordering matters and is right: baseUrl first (cheap, exact),
  model-name fallback second. The inline comment at lines 28-33 names
  the issue (#3613) and explains *why* model-name detection is the right
  proxy for the content-format constraint, which is unusually good for
  a 9-line patch.

- **Test coverage is precise.**
  `provider/deepseek.test.ts:52-87` covers three angles:
  1. The existing baseUrl-negative case is updated to also pin
     `model: 'gpt-4o'` so the new rule doesn't accidentally satisfy
     it via the test's spread-default `deepseek-chat` (lines 52-62).
     This is the kind of subtle test-fixture interaction that catches
     authors off-guard; the author saw it and fixed it.
  2. The sglang scenario (lines 64-74) — `https://my-sglang.example.com`
     baseUrl + `deepseek-v4-pro` model — directly mirrors the bug report.
  3. Case-insensitive matching (lines 76-86) — `DeepSeek-R1` on a vllm
     baseUrl. Closes the obvious "what about caps" follow-up.

- **One forward-compat concern worth a comment, not blocking.** A
  substring match on `"deepseek"` will also match a hypothetical model
  name like `not-deepseek-compatible-v2` from an unrelated vendor. The
  blast radius is small (the only behaviour gated on this is the
  content-array → string flattening, which is harmless for any model
  that *can* accept the array form), but a `\bdeepseek` or
  `^|[/_-]deepseek` boundary check would be marginally safer. The
  practical answer is probably "no — vendors don't ship `xdeepseek`",
  so this is a nit at most.

- **`buildRequest` flattening itself unchanged.** I traced the
  flattening path (`buildRequest` walks `messages[*].content`, joins
  text parts, and writes back a string) and it doesn't touch this PR.
  Good — the change is purely in the *gate*, not the *behaviour*, so
  the blast radius is well-contained.

## Verdict

`merge-after-nits`

(One-line `\b`-boundary upgrade for the model-name match, or at minimum
a comment noting the substring choice is intentional.)
