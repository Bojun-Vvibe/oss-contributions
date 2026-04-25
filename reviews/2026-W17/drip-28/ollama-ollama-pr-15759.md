# PR #15759 — x/mlxrunner: recognise mlx-lm plural aux naming at load time

- Repo: ollama/ollama
- Head SHA: 620bcf80228ce31ab9ac1964c03f4627428f1009
- Files touched: 7 (+310 / -27)

## Summary

Teaches the MLX runner's tensor loading path to accept mlx-lm's
sibling-plural quant-aux naming (`<module>.scales` /
`<module>.biases`) alongside Ollama's existing dot-child singular
naming (`<weight>.scale` / `<weight>.bias`). Strictly
name-recognition at load time — quant-parameter resolution
(`TensorQuantParams`, `ResolveLinearQuantParams`) and forward-
pass code are unchanged. Without this, layers in mlx-lm-imported
blobs whose aux tensors stayed in the plural convention get
constructed as raw float dense linears, silently loading U32-
packed weights as floats. Supersedes #15743.

## Specific findings

- `x/mlxrunner/model/linear.go:60-79` —
  `MakeLinearLayer` falls back to `tensors[path+".scales"]` when
  `path+".weight_scale"` is missing, then to
  `tensors[path+".biases"]` for qbiases. Same shape applies in
  `x/mlxrunner/model/embedding.go:23-41` for embeddings.
- `x/mlxrunner/model/root.go:200-211` —
  `mainTensorNames` skip-list grows from
  `{".scale", ".bias"}` to also include `.scales`/`.biases`,
  so the plural aux names don't get treated as main tensors and
  iterated through the dense-construction path.
- `x/mlxrunner/runner.go:87-99` (and onward to ~155) —
  `loadTensorsFromManifest` Phase-2 remap normalises both
  conventions to the canonical `<weight>_scale` form. Per the
  PR body and the new `runner_test.go` test
  `TestNormaliseAuxNames_PluralAndSingular`, **singular wins
  deterministically** when both target the same canonical key.
  That's the key correctness invariant — without it, Go map
  iteration order would be the tiebreaker.
- New tests:
  - `model/linear_test.go:1-44` —
    `TestMakeLinearLayer_MLXLMSiblingQuantized` builds a fake
    plural-named tensor map and asserts the result is
    `*nn.QuantizedLinear` with the right tensors wired (the
    failure mode the PR exists to prevent: silent fallthrough
    to `*nn.Linear`).
  - `model/embedding_test.go:79-110` —
    `TestMakeEmbeddingLayerQuantizedMLXLMSibling` covers the
    same shape for embeddings.
  - `runner_test.go:1-127` (new) —
    `TestNormaliseAuxNames_PluralAndSingular` covers six
    cases: singular dot-child, mlx-lm sibling plural, mixed,
    dense bias passthrough, plural bias passthrough, and the
    explicit singular-wins precedence case.

## Risks / nits

- The `MakeLinearLayer` / `MakeEmbeddingLayer` fallbacks are
  correctly *fallback only* — the singular form is checked
  first, so existing Ollama-native blobs are unaffected. The
  remap in `runner.go` does the same (singular first, plural
  fallback), with the deterministic-singular-wins tiebreak
  baked into the test suite.
- The PR's scope guard ("strictly about name recognition")
  is the right call — it cleanly separates the *recognition*
  fix from the downstream *correctness* fix on Qwen 3.6 MoE
  forward-pass. That second PR is the dangerous one;
  reviewer should confirm it's tracked and won't quietly
  block on this.
- `TestNormaliseAuxNames_PluralAndSingular` doesn't appear
  to test what happens when **both** `<path>.weight_scale`
  AND `<path>.scales` exist for the *same* path on disk
  (i.e. a malformed safetensors blob with both conventions
  present for the same module). The PR body says "singular
  wins deterministically when both target the same canonical
  key"; reviewer should confirm the test asserts that exact
  case (the description names "deterministic singular-wins
  precedence" but I'd want to see the test body to be sure).
- `mainTensorNames` now excludes any tensor whose name ends
  in `.scales` or `.biases`. There's a vanishingly small
  chance of a real model weight legitimately named
  `something.biases` (e.g. some bizarre custom op). For the
  models targeted by mlx-lm this isn't a concern, but a
  comment noting the assumption would help the next
  maintainer.

## Verdict

**merge-after-nits** — High-quality, scope-disciplined fix.
The fallback pattern is correct, the deterministic-tiebreak
addresses the only real correctness concern, and the test
matrix covers the relevant shapes. Confirm the
both-conventions-present test case is explicit, and add a
comment on `mainTensorNames` about the `.biases`/`.scales`
suffix exclusion.

## What I learned

When two tools serialize the same logical concept under
different key conventions, the safest fix is to normalise to
a third *canonical* name at the boundary, with a deterministic
tiebreak rule for the rare both-present case. The
unsafe-but-tempting alternative — picking convention A or B
based on iteration order — silently introduces a
non-reproducible failure that only surfaces under the right
hash seed. Pinning the tiebreak with a unit test that names
the rule ("singular wins") is the difference between a fix
that holds and one that drifts.
