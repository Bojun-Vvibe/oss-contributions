# ollama/ollama PR #15761 — Baseline test coverage for quant metadata scanning

- **Repo:** ollama/ollama
- **PR:** [#15761](https://github.com/ollama/ollama/pull/15761)
- **Head SHA:** `26b0e2c4695b2a59fee0924e2c7ab972152009af`
- **Author:** jodagreyhame
- **Size:** +745 / −27 across 9 files (supersedes #15745)
- **Reviewer:** Bojun (drip-27)

## Summary

Adds baseline Go test coverage for the `x/mlxrunner/model` package's
quant metadata scanning, plus a small set of fixes to the production
code surfaced while writing the tests. Also recognises the mlx-lm
sibling-plural aux naming (`<path>.scales` / `<path>.biases`) in
`MakeLinearLayer` and `MakeEmbeddingLayer` as an alternative to the
Ollama-native dot-child singular form (`<path>.weight_scale` /
`<path>.weight_qbias`). PR body explicitly calls out that the previous
review iteration (Codex bot) caught one P1 in the test setup
(`int8_groupSize64` had wrong shapes) and that two further latent test
bugs surfaced once `skipIfNoMLX` wasn't masking them.

## Key changes

### `x/mlxrunner/model/linear.go:44–58` — doc + naming alternative

```go
// Two scale/qbias naming conventions are recognised:
//   - Ollama-native dot-child singular: "<path>.weight_scale" / "<path>.weight_qbias"
//   - mlx-lm sibling plural: "<path>.scales" / "<path>.biases"
```

The doc comment now spells out the two conventions and *why* both are
recognised — community MLX safetensors imported via
`ollama create --experimental` may emit either, depending on whether
the source used `mx.nn.quantize` (mlx-lm/LM Studio) or the
Ollama-native conversion path.

### `x/mlxrunner/model/linear.go:69–76` — fallback lookup

```go
scales := tensors[path+".weight_scale"]
if scales == nil {
    scales = tensors[path+".scales"]
}
if scales != nil {
    qbiases := tensors[path+".weight_qbias"]
    if qbiases == nil {
        qbiases = tensors[path+".biases"]
    }
    ...
}
```

The Ollama-native key is checked first (preserving existing behaviour
on Ollama-native blobs), with the plural form as a fallback. This is
the right precedence — anything that previously worked still loads via
the same code path, and the fallback only fires for blobs that *only*
have the plural form.

### `x/mlxrunner/model/embedding.go:23–40` — same pattern, mirrored

The embedding layer applies the identical fallback. Worth noting both
files now have the same six-line "check singular, fall back to plural"
shape — a small private helper would deduplicate, but the lookups are
on different tensor maps and called once each, so the duplication is
arguably clearer than abstraction here.

### `x/mlxrunner/model/embedding_test.go:80–110` — new test

`TestMakeEmbeddingLayerQuantizedMLXLMSibling` builds a quantized
`denseWeight` of shape `[2, 64]`, calls `mlx.Quantize(...)` to get
`(qw, scales, qbiases)`, builds the tensor map under the *plural*
naming (`model.embed_tokens.scales` / `.biases`), and asserts the
returned layer is a `*nn.QuantizedEmbedding` with the expected
`(GroupSize=64, Bits=4, Mode="affine")`. Pairs with the matching
linear-layer test in `linear_test.go`.

### Quant param test fixes (per PR body)

The PR body lists three test fixes the author landed *on top of* the
superseded #15745:

1. `quant_test.go:124` — `int8_groupSize64` was using shapes
   `[4,4]/[4,1]`, which resolves to `groupSize4 == 32` and early-returns
   `(32, 4)`, never reaching the `groupSize8 == 64` branch the test
   name advertised. Corrected to `[4, 16]/[4, 1]` so the test actually
   exercises the `groupSize8 == 64` arm and asserts `(64, 8, true)`.
2. `TestMakeLinearLayer_OllamaNativeQuantized` — was using `affine`
   mode + shapes that triggered shape inference, overwriting the
   defaults the test intended to verify. Switched to shapes that
   skip inference.
3. A third latent bug in the same vein, surfaced once `skipIfNoMLX`
   wasn't hiding it.

## What's good

- The naming-fallback fix is the smallest possible change that solves
  the actual interop problem ("mlx-lm and LM Studio safetensors don't
  load"). No flag, no migration, no opt-in — just "recognise the
  alternate spelling".
- The author's response to the prior `Codex` review feedback is
  visible in the diff: the rejected setup with bad shapes is gone and
  replaced with shapes that actually exercise the intended branch.
  That's exactly the right thing to do with a P1 review comment, and
  the PR body cites the line (`quant_test.go:124`) that was wrong.
- The two latent test bugs surfaced by *running with MLX active*
  (rather than via the bot's `skipIfNoMLX` path) are a great example
  of why CI matrix coverage matters — the bot couldn't see them, but
  a human running locally with MLX could.
- Doc comments on `MakeLinearLayer` and `MakeEmbeddingLayer` now
  explicitly enumerate both conventions and link the rationale. This
  is the kind of documentation that pays back the next time someone
  has to debug "why doesn't this safetensor load?".

## Concerns

1. The lookup pattern is duplicated verbatim across `linear.go` and
   `embedding.go`. A `func resolveAuxTensors(tensors map[string]*mlx.Array,
   path string) (scales, qbiases *mlx.Array)` helper would deduplicate
   ~12 lines and centralise the precedence rule. Worth a follow-up if
   a third call site appears (e.g. attention projections).

2. The fallback is silent — if a blob uses the plural naming, nothing
   logs that fact. For diagnosing "why is my model loading slowly /
   weirdly?" it would be useful to emit a `slog.Debug` at first
   fallback hit, e.g. `slog.Debug("mlxrunner: using mlx-lm sibling
   aux naming", "path", path)`. Cheap to add, very useful when a
   user reports an interop issue.

3. The test for the linear layer (`linear_test.go`, new file) and
   the new embedding test both call `skipIfNoMLX(t)`. If a future
   CI lane is added that *does* have MLX, the tests will run and
   exercise the fallback path. If no such lane exists, the
   fallback is implicitly only tested by humans running locally.
   Worth confirming the CI matrix actually has at least one MLX-
   enabled job, or the regression will only surface on user reports.

4. The PR is +745 / −27 across 9 files but the production-code delta
   is ~12 lines (the two fallback blocks). The bulk is test coverage
   plus doc comments — generally a good ratio, but worth flagging
   that the actual behavioural change is small enough to merit
   verification by the test names alone (which read clearly).

## Risk

Low. The fallback only fires when the Ollama-native key is missing,
so existing Ollama-native blobs follow the exact same code path. The
test fixes correct latent assertions that were passing for the wrong
reason — which is strictly an improvement to the suite.

## Verdict

**merge-after-nits**

The naming-fallback change is correct and minimal, the doc updates
are useful, and the test fixes restore real assertion coverage. The
silent-fallback `slog.Debug` (concern 2) is worth landing before
merge — it's three lines of code and saves future support time. The
helper extraction (concern 1) can wait for a third call site.

## What I learned

"Check singular, fall back to plural" with the Ollama-native form
*first* is exactly the right precedence for a backwards-compatible
interop fix: it preserves the existing fast path bit-for-bit, and
the new path is only taken when the old one would have failed
anyway. The test author's response to the prior bot review (fixing
a P1 about wrong shapes by actually changing the shapes, not by
changing the assertion) is also worth noting — that's the
"strengthen the test, don't weaken it" instinct that keeps suites
useful over time.
