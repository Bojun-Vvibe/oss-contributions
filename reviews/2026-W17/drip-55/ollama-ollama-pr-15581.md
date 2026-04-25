# ollama/ollama PR #15581 — ggml-metal: fix mixed bf16/f16 cooperative tensor operand order

@c95a23ac · base `main` · +37/-4 · author `Horacehxw`

## Summary
Fixes a Metal-shader compile failure on Apple M5 / Metal 4 systems for mixed `bf16` / `f16` matmuls. The destination cooperative-tensor type was being instantiated with the operand order `(decltype(tA), decltype(tB), float)`, but the matmul itself calls `mm.run(sB, sA, …)` — so when `tA` and `tB` had different element types, the static_assert `__is_same_v<bfloat, half>` fired with `"Input types must match cooperative tensor types"`.

## What changed
- `ml/backend/ggml/ggml/src/ggml-metal/ggml-metal.metal:9152, 9537` — both `kernel_mul_mm` and `kernel_mul_mm_id` swap operand order:
  ```cpp
  - auto cT = mm.get_destination_cooperative_tensor<decltype(tA), decltype(tB), float>();
  + auto cT = mm.get_destination_cooperative_tensor<decltype(tB), decltype(tA), float>();
  ```
- `ml/backend/ggml/ggml/src/ggml-metal/ggml-metal-embed.metal:11974, 12359` — same two changes in the embedded copy of the Metal shader.
- `llama/patches/0037-ggml-metal-match-cooperative-tensor-operand-order.patch` (new, 33 lines) — the canonical upstream patch series, which keeps the vendored ggml in lockstep with the change pushed up to llama.cpp.

## Key observations
- This is a one-character fix repeated four times across two generated files plus the patch file. The right shape: the change is the same identifier swap everywhere, the patch file is the upstream-of-record, and the embedded `.metal` plus the in-tree `.metal` both have to be updated since the embed file is generated/checked-in.
- The diagnosis in the PR description ("the matmul call uses `(sB, sA)` but the destination type is `(tA, tB)`") is precise and matches the static_assert message; the fix is consistent with that diagnosis.
- No regression test, but for a Metal shader compile-time assert there really isn't anything to unit-test in Go — the test surface is "does it compile on Apple M5 with mixed bf16/f16". A maintainer with M5 hardware should confirm; CI Mac runners may not catch this.
- Author notes the patch file is the upstream-of-record. Recommend the maintainer cross-check that an equivalent change has been (or will be) submitted to upstream llama.cpp / ggml so this patch eventually drops out of the local series.

## Risks/nits
- Behavior change is "compiles where it didn't" — zero risk to correctness on systems that already worked, since the underlying matmul math doesn't change.
- The two updated files are mirrors of each other; a regeneration script (if one exists) should also be run to confirm they don't drift in the next sync.

**Verdict: merge-as-is**
